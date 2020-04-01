---
title: Android P SharedPreferences解析
date: 2018-12-09 11:42:06
tags: Android
---
`SharedPreferences` 作为`Android`常用的存储方式，它适合存储小数据量，特别是键值对数据。
原理的文章可以参考：
[深入理解Android中的SharedPreferences](https://blog.csdn.net/u012619640/article/details/50940074)
[SharedPreferences源码解析](https://juejin.im/post/5ad97691518825671c0e19d9)
细节绘图就不上了，这里会从Android 9.0的源码去重新捋一遍过程，也算是自我的学习review。

# SharedPreferences 接口
`SharedPreferences`是一个接口，其中包含了`Editor`接口与`OnSharedPreferenceChangeListener`接口

<!--more-->
```java
public interface SharedPreferences {
    public interface OnSharedPreferenceChangeListener {
        void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
    }
    public interface Editor {
        Editor putString(String key, @Nullable String value);
        Editor putStringSet(String key, @Nullable Set<String> values);
        Editor putInt(String key, int value);
        Editor putLong(String key, long value);
        Editor putFloat(String key, float value);
        Editor putBoolean(String key, boolean value);
        Editor remove(String key);
        Editor clear();

        /**
         * Commit your preferences changes back from this Editor to the
         * {@link SharedPreferences} object it is editing.  This atomically
         * performs the requested modifications, replacing whatever is currently
         * in the SharedPreferences.
         *
         * <p>Note that when two editors are modifying preferences at the same
         * time, the last one to call commit wins.
         *
         * <p>If you don't care about the return value and you're
         * using this from your application's main thread, consider
         * using {@link #apply} instead.
         *
         * @return Returns true if the new values were successfully written
         * to persistent storage.
         */
        boolean commit();

        /**
         * Commit your preferences changes back from this Editor to the
         * {@link SharedPreferences} object it is editing.  This atomically
         * performs the requested modifications, replacing whatever is currently
         * in the SharedPreferences.
         *
         * <p>Note that when two editors are modifying preferences at the same
         * time, the last one to call apply wins.
         *
         * <p>Unlike {@link #commit}, which writes its preferences out
         * to persistent storage synchronously, {@link #apply}
         * commits its changes to the in-memory
         * {@link SharedPreferences} immediately but starts an
         * asynchronous commit to disk and you won't be notified of
         * any failures.  If another editor on this
         * {@link SharedPreferences} does a regular {@link #commit}
         * while a {@link #apply} is still outstanding, the
         * {@link #commit} will block until all async commits are
         * completed as well as the commit itself.
         *
         * <p>As {@link SharedPreferences} instances are singletons within
         * a process, it's safe to replace any instance of {@link #commit} with
         * {@link #apply} if you were already ignoring the return value.
         *
         * <p>You don't need to worry about Android component
         * lifecycles and their interaction with <code>apply()</code>
         * writing to disk.  The framework makes sure in-flight disk
         * writes from <code>apply()</code> complete before switching
         * states.
         *
         * <p class='note'>The SharedPreferences.Editor interface
         * isn't expected to be implemented directly.  However, if you
         * previously did implement it and are now getting errors
         * about missing <code>apply()</code>, you can simply call
         * {@link #commit} from <code>apply()</code>.
         */
        void apply();
    }

    Map<String, ?> getAll();
    String getString(String key, @Nullable String defValue);
    Set<String> getStringSet(String key, @Nullable Set<String> defValues);
    int getInt(String key, int defValue);
    long getLong(String key, long defValue);
    float getFloat(String key, float defValue);
    boolean getBoolean(String key, boolean defValue);
    boolean contains(String key);
    Editor edit();
    void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
    void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
}
```
从接口中，我们大概了解`SharedPreferences`需要包含的东西。其中主要的`commit`和`apply`，保留了注释，是我们关键要注意的方法。

# 初始化
## Context.getSharedPreferences
```java
public abstract SharedPreferences getSharedPreferences(String name, @PreferencesMode int mode);
```
Android P(9.0)中，该方法声明为虚函数，它的实现者是`ContextImpl`

## ContextImpl.getSharedPreferences
```java
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;
private ArrayMap<String, File> mSharedPrefsPaths;
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
    // 小于Android 19(4.4)时，默认文件名是null.xml
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }

    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            file = getSharedPreferencesPath(name);
            mSharedPrefsPaths.put(name, file);
        }
    }
    return getSharedPreferences(file, mode);
}

@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            // 模式检查
            checkMode(mode);
            // Android O（8.0）以上，检查身份
            if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                if (isCredentialProtectedStorage()
                        && !getSystemService(UserManager.class)
                                .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                    throw new IllegalStateException("SharedPreferences in credential encrypted "
                            + "storage are not available until after user is unlocked");
                }
            }
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}
    
// Android N（7.0）以上，不支持MODE_WORLD_READABLE和MODE_WORLD_WRITEABLE模式。
private void checkMode(int mode) {
    if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.N) {
        if ((mode & MODE_WORLD_READABLE) != 0) {
            throw new SecurityException("MODE_WORLD_READABLE no longer supported");
        }
        if ((mode & MODE_WORLD_WRITEABLE) != 0) {
            throw new SecurityException("MODE_WORLD_WRITEABLE no longer supported");
        }
    }
}
    
private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    final String packageName = getPackageName();
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    return packagePrefs;
}
```
从`getSharedPreferencesCacheLocked`方法，注意`sSharedPrefsCache`是一个`static`对象，保存了（`packageName` -》`ArrayMap<File, SharedPreferencesImpl>`）键值对，而`mSharedPrefsPaths`保存了(`name`-》`File`)的键值对。所以在整个应用执行过程中，一个包名只对应一个`ArrayMap`，而对于不同的`name`，可以对应不同的`SharedPreferencesImpl`对象。

在7.0版本以上，对于安全的检测更加严格，增加了新特性`Direct boot`(与判断`isCredentialProtectedStorage`相关，文章参考：[Android 7.0 新特性--Direct Boot](https://blog.csdn.net/unicorn97/article/details/53053601))，并且加上`UserManager`的`isUserUnlockingOrUnlocked`方法判断。**简单来说，如果用户手机没有解锁，则不会进行sp的读取操作**。

## new SharedPreferencesImpl
```java
private final File mFile;
private final File mBackupFile;
private final int mMode;
private final Object mLock = new Object();
private Map<String, Object> mMap;
private Throwable mThrowable;

SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}

static File makeBackupFile(File prefsFile) {
    return new File(prefsFile.getPath() + ".bak");
}
```
具体的实现，在`SharedPreferencesImpl`类中。初始化过程中，`mBackupFile`是.bak后缀的文件。我们继续看`startLoadFromDisk`的实现。

## startLoadFromDisk
```java
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```
开启了新线程执行`loadFromDisk`。

## loadFromDisk
```java
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        // 如果.bak文件存在，说明上一次写文件没有成功，把数据恢复
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    // Debugging
    if (mFile.exists() && !mFile.canRead()) {
        Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
    }

    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                new FileInputStream(mFile), 16 * 1024);
                // 读xml文件
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;

        // It's important that we always signal waiters, even if we'll make
        // them fail with an exception. The try-finally is pretty wide, but
        // better safe than sorry.
        try {
            if (thrown == null) {
                if (map != null) {
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
            // In case of a thrown exception, we retain the old map. That allows
            // any open editors to commit and store updates.
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            mLock.notifyAll();
        }
    }
}
```
这段代码很简单，如果有`.bak`文件，则把该文件替换掉`xxx.xml`文件，然后从这个文件中读取map数据，赋值给`mMap`。结束之后通过`mLock.notifyAll`唤醒其他线程

# 读数据
我们先看读的操作过程。`getXXX()`的逻辑是类似的，我们只看`getString`就好。
```java
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```
加锁调用`awaitLoadedLocked`，由于初始化`SharedPreferencesImpl`时，`loadFromDisk`是在子线程执行的，在这里加锁等待直到加载成功。
**因此，如果在UI线程读取数据，等待时间过长的话，有可能引起anr**


# 写数据
`Editor`是负责处理数据的接口，我们继续看`SharedPreferencesImpl.editor()`的实现。
## SharedPreferencesImpl.edit
```java
public Editor edit() {
    // TODO: remove the need to call awaitLoadedLocked() when
    // requesting an editor.  will require some work on the
    // Editor, but then we should be able to do:
    //
    // context.getSharedPreferences(..).edit().putString(..).apply()
    // ... all without blocking.
    synchronized (mLock) {
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```
等待数据读取成功，然后返回`new EditorImpl()`对象。

## putString
我们以`putXXX()`都是相似的，我们以`putString()`为例
```java
public final class EditorImpl implements Editor {
    private final Object mEditorLock = new Object();
    private final Map<String, Object> mModified = new HashMap<>();
    public Editor putString(String key, @Nullable String value) {
        synchronized (mEditorLock) {
            mModified.put(key, value);
            return this;
        }
    }
    ...
}
```
`putXXX`也是一个同步操作，往`mModified`放数据。我们来看关键的方法有什么区别：`commit()`和`apply()`。

# 提交数据：commit
```java
public boolean commit() {
    MemoryCommitResult mcr = commitToMemory();

    // commit同步提交数据
    SharedPreferencesImpl.this.enqueueDiskWrite(
            mcr, null /* sync write on this thread okay */);
    try {
        // 等待过程结束
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        // debug code
    }
    notifyListeners(mcr);
    // 返回执行结果
    return mcr.writeToDiskResult;
}
```
主要有两个步骤，一个是`commitToMemory`生成`MemoryCommitResult`对象，记录内存操作。然后通过`enqueueDiskWrite`对处理这个对象。

## commitToMemory
```java
// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;

    synchronized (SharedPreferencesImpl.this.mLock) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        // 是否正在提交的过程。
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            // 处于正在提交的过程，mMap的数据不能被修改，只能再拷贝一份
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            // 有监听者，也拷贝
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        // 确保不让其他线程修改数据
        synchronized (mEditorLock) {
            boolean changesMade = false;

            // 这是清除标记，调用Editor.clear()时设置该标记为true
            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    // 清除从mMap读取的数据，即从xml中读取的数据
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }

            // 把mModified中键值对读入mapToWriteToDisk。
            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                // this值，是调用Editor.remove的时候设置的。
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    // 从现有数据中，移除对象k
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }

                changesMade = true;
                if (hasListeners) {
                    // 记录有数据变动的key
                    keysModified.add(k);
                }
            }

            mModified.clear();

            if (changesMade) {
                // 变更版本号
                mCurrentMemoryStateGeneration++;
            }

            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners, mapToWriteToDisk);
}
```
需要留意`mMap`和`mModified`的区别，`mMap`是从`.xml`文件中读出来的键值对，`mModified`是editor中放进去的键值对。这个过程，是把`mModified`中的键值对，转存到`mapToWriteToDisk`中去。其中`EditorImpl`中会保存一个`clear`标记，记录是否需要清除原来的数据。如果一个键值对的value是this（SharedPreferencesImpl）或者是null那么表示将此键值对删除。代码的使用如下：
```java
public Editor clear() {
    synchronized (mEditorLock) {
        mClear = true;
        return this;
    }
}

public Editor remove(String key) {
    synchronized (mEditorLock) {
        mModified.put(key, this);
        return this;
    }
}
```
把数据都读取到`mapToWriteToDisk`之后，记录数据有变化的键值对，升级针对本次编辑的版本号，生成`MemoryCommitResult`对象。

## new MemoryCommitResult
```java
private static class MemoryCommitResult {
    final long memoryStateGeneration;
    @Nullable final List<String> keysModified;
    @Nullable final Set<OnSharedPreferenceChangeListener> listeners;
    final Map<String, Object> mapToWriteToDisk;
    final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

    @GuardedBy("mWritingToDiskLock")
    volatile boolean writeToDiskResult = false;
    boolean wasWritten = false;

    private MemoryCommitResult(long memoryStateGeneration, @Nullable List<String> keysModified,
    @Nullable Set<OnSharedPreferenceChangeListener> listeners,
    Map<String, Object> mapToWriteToDisk) {
        this.memoryStateGeneration = memoryStateGeneration;
        this.keysModified = keysModified;
        this.listeners = listeners;
        this.mapToWriteToDisk = mapToWriteToDisk;
    }

    void setDiskWriteResult(boolean wasWritten, boolean result) {
        this.wasWritten = wasWritten;
        writeToDiskResult = result;
        // 释放共享锁
        writtenToDiskLatch.countDown();
    }
}
```
对象记录了`commitToMemory`过程中的结果项，包括了：修改版本号，有变动的keys，监听者，待存储的键值对数据。

## enqueueDiskWrite
```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    // commit()时，isFromSyncCommit = true
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            // 与commit相对应。
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }
    // apply时调用。
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```
如果是`commit`调用，将会在**本线程**执行同步写操作。**所以注意commit操作不要再UI线程调用**。

## writeToFile
```java
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    // ... 省略debug code，跟耗时计算相关
    // Rename the current file so it may be used as a backup during the next read
    if (fileExists) {
        boolean needsWrite = false;

        // Only need to write if the disk state is older than this commit
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
            if (isFromSyncCommit) {
                // commit时进来
                needsWrite = true;
            } else {
                synchronized (mLock) {
                    // No need to persist intermediate states. Just wait for the latest state to
                    // be persisted.
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }

        if (!needsWrite) {
            // 不需要写，则结束过程。
            mcr.setDiskWriteResult(false, true);
            return;
        }
        
        boolean backupFileExists = mBackupFile.exists();
        if (!backupFileExists) {
            // .bak不存在，则把xml重命名为.bak
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            // 如果.bak存在，则目标xml可以删除，下面再重写文件
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);
        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        // 写文件
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
        FileUtils.sync(str);
        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                // 记录时间戳和文件大小
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }

        // Writing was successful, delete the backup file if there is one.   xml文件写成功了，就删除.bak文件
        mBackupFile.delete();
        mDiskStateGeneration = mcr.memoryStateGeneration;
        mcr.setDiskWriteResult(true, true);
        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }

    // Clean up an unsuccessfully written file
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```
这里如果写成功，则把`.bak`文件删除。

## notifyListeners
执行结束之后，通知监听者。
```java
private void notifyListeners(final MemoryCommitResult mcr) {
    if (mcr.listeners == null || mcr.keysModified == null ||
        mcr.keysModified.size() == 0) {
        return;
    }
    if (Looper.myLooper() == Looper.getMainLooper()) {
        for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
            final String key = mcr.keysModified.get(i);
            for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                if (listener != null) {
                     listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                }
            }
        }
    } else {
        // Run this function on the main thread.
        ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
    }
}
```
从主线程回调通知给监听者。

`commit`过程中，需要注意的点：
1、在当前线程同步执行文件的写操作，调用时注意当前是否处于UI线程，避免发生anr。
2、监听者的回调，是在主线程执行的。

# 提交数据：apply
```java
public void apply() {
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    // 等待writeToFile执行完
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };

    // 添加到未完成任务队列中，会影响到主线程。
    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```

## QueuedWork.addFinisher
```java
private static final LinkedList<Runnable> sFinishers = new LinkedList<>();
public static void addFinisher(Runnable finisher) {
    synchronized (sLock) {
        sFinishers.add(finisher);
    }
}
```
这是一个全局调用。`sFinishers`是静态变量，意味着别的地方，也有可能使用到这个变量。

与`commit`过程不同，`enqueueDiskWrite`传的参数，带上了`postWriteRunnable`。会执行到`QueuedWork.queue`中。

`QueuedWork`需要等待`awaitCommit`执行，才会把该任务移除。`postWriteRunnable`什么时候执行呢？从`enqueueDiskWrite`中，知道调用时机，是在`writeToFile`完成后执行的。而`writeToDiskRunnable`任务，是在`QueuedWork`中排队等待执行。见[enqueueDiskWrite](#enqueueDiskWrite)

## QueuedWork.queue
```java
private static final LinkedList<Runnable> sWork = new LinkedList<>();
public static void queue(Runnable work, boolean shouldDelay) {
    Handler handler = getHandler();

    synchronized (sLock) {
        sWork.add(work);

        // sCanDelay为true, DELAY = 100
        if (shouldDelay && sCanDelay) {
            handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
        } else {
            handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
        }
    }
}
```
把任务延迟100ms发给`Handler`执行。

## QueuedWork.getHandler
```java
private static Handler getHandler() {
    synchronized (sLock) {
        if (sHandler == null) {
            HandlerThread handlerThread = new HandlerThread("queued-work-looper",
            Process.THREAD_PRIORITY_FOREGROUND);
            handlerThread.start();

            sHandler = new QueuedWorkHandler(handlerThread.getLooper());
        }
        return sHandler;
    }
}

private static class QueuedWorkHandler extends Handler {
    static final int MSG_RUN = 1;

    QueuedWorkHandler(Looper looper) {
        super(looper);
    }

    public void handleMessage(Message msg) {
        if (msg.what == MSG_RUN) {
            processPendingWork();
        }
    }
}
```
`getHandler`是创建了一个后台线程，通过`HandlerThread`来顺序处理任务。

## QueueWork.processPendingWork
```java
/**
 * Used to make sure that only one thread is processing work items at a time. This means that
 * they are processed in the order added.
 *
 * This is separate from {@link #sLock} as this is held the whole time while work is processed
 * and we do not want to stall the whole class.
 */
private static Object sProcessingWork = new Object();
private static void processPendingWork() {
    synchronized (sProcessingWork) {
        LinkedList<Runnable> work;

        synchronized (sLock) {
            work = (LinkedList<Runnable>) sWork.clone();
            sWork.clear();

            // Remove all msg-s as all work will be processed now
            getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
        }

        if (work.size() > 0) {
            for (Runnable w : work) {
                w.run();
            }
        }
    }
}
```
这个方法，在`queued-work-looper`线程中，把`sWork`队列中的任务都拿出来循环执行，清空了`Handler`中的队列。

## processPendingWork其他调用时机---------waitToFinish
除了`QueuedWorkHandler`调用这个方法，在`waitToFinish`中也会被调用到。
```java
/**
 * Trigger queued work to be processed immediately. The queued work is processed on a separate
 * thread asynchronous. While doing that run and process all finishers on this thread. The
 * finishers can be implemented in a way to check weather the queued work is finished.
 *
 * Is called from the Activity base class's onPause(), after BroadcastReceiver's onReceive,
 * after Service command handling, etc. (so async work is never lost)
 */
public static void waitToFinish() {
    boolean hadMessages = false;
    Handler handler = getHandler();

    synchronized (sLock) {
        if (handler.hasMessages(QueuedWorkHandler.MSG_RUN)) {
            // Delayed work will be processed at processPendingWork() below
            handler.removeMessages(QueuedWorkHandler.MSG_RUN);
        }

        // We should not delay any work as this might delay the finishers
        sCanDelay = false;
    }

    StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
    try {
        processPendingWork();
    } finally {
        StrictMode.setThreadPolicy(oldPolicy);
    }

    try {
        while (true) {
            Runnable finisher;
            synchronized (sLock) {
                finisher = sFinishers.poll();
            }
            if (finisher == null) {
                break;
            }
            finisher.run();
        }
    } finally {
        sCanDelay = true;
    }
}
```
从注释中看，`waitToFinish`会从Activity onPause()/BroadcastReceiver onReceive()/Service 进去，执行`processPendingWork`之后，再死循环把`sFinishers`中任务执行完。<font color=red>**注意：Activity onPause是主线程执行的，这时候拿出sWork中的任务，这其中就包含了writeToDiskRunnable，是存在IO操作的。如果apply任务过多，数据量大，很可能引发anr**</font>

## 调用waitToFinish的地方有哪些？
基本集中在`ActivityThread.java`类中，即主线程执行。
- handleServiceArgs
- handleStopService
- handlePauseActivity
- handleStopActivity
- handleSleeping

我们挑个handleStopActivity看看。
```java
public void handleStopActivity(IBinder token, boolean show, int configChanges,
            PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
    final ActivityClientRecord r = mActivities.get(token);
    r.activity.mConfigChangeFlags |= configChanges;

    final StopInfo stopInfo = new StopInfo();
    performStopActivityInner(r, stopInfo, show, true /* saveState */, finalStateRequest,
                reason);

    if (localLOGV) Slog.v(
        TAG, "Finishing stop of " + r + ": show=" + show
        + " win=" + r.window);

    updateVisibility(r, show);

    // Make sure any pending writes are now committed.
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }

    stopInfo.setActivity(r);
    stopInfo.setState(r.state);
    stopInfo.setPersistentState(r.persistentState);
    pendingActions.setStopInfo(stopInfo);
    mSomeActivitiesChanged = true;
}
```
![](/images/SharedPreferences/handleStopActivity.jpg)

从模拟器中断点，确实是从主进程进去的。

# 总结
sp在实现上，还是会有些瑕疵，如果使用不当，会引发稳定性问题。下面给出使用的建议。
- sp不要存储大数据量的键值对
- 读sp数据时，尽量不在主线程中读取，因为读取过程前会锁住线程，等待数据文件读取完毕。
- commit是在调用线程中执行文件读写，apply是提交到子线程执行。因此，使用commit时，注意不要在主线程调用，apply则没这个要求。
- 不要频繁调用apply，应该一次edit之后，处理完数据之后，再apply。apply的调用间隔不应太短。
- 不要使用MODE_MULTI_PROCESS。

