---
title: AsyncTask
date: 2018-11-28 18:12:49
tags:
- 源码解析
---

关于AsyncTask的解析文章，网上有很多：
https://blog.csdn.net/liuhe688/article/details/6532519
我这里对于怎么使用就不赘述了，直接进到源码中看（API 28）

# 从调用开始
```java
new AsyncTask<Void, Void, Void>() {
    @Override
    protected Void doInBackground(Void... voids) {
    	return null;
    }
}.execute();
```

<!-- more -->
## AsyncTask构造函数
```java
    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        this((Looper) null);
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Handler handler) {
        this(handler != null ? handler.getLooper() : null);
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);	// 创建一个handler，实际是一个InternalHandler对象
        /**
        private static Handler getMainHandler() {
            synchronized (AsyncTask.class) {
            if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
            }
        }
        **/

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
        /**
        // WorkerRunnable实现了Callable接口
        private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
            Params[] mParams;
        }
        
        // 这个接口与Runnable作用一样，只是Callable可以返回对象，以及抛出异常
        public interface Callable<V> {
            V call() throws Exception;
        }
        **/

        // 最终的任务交由FutureTask来装载
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
        
        /**
        public FutureTask(Callable<V> callable) {
            if (callable == null)
                throw new NullPointerException();
            this.callable = callable;
            this.state = NEW;       // ensure visibility of callable
        }
        **/
    }
```
初始化了`Handler`，`FutureTask`对象。`FutureTask`对象，保存了`WorkerRunnable`。

## AsyncTask.execute()
```java
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();		// 这里会回调onPreExecute()。

        mWorker.mParams = params;
        exec.execute(mFuture);		// exec是sDefaultExecutor，是一个SerialExecutor对象

        return this;
    }
```
判断修改当前任务状态，并由自定义的任务管理器执行任务。

## 线程池调用：SerialExecutor.execute
```java
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            // 外面封装了一层，由ArrayDeque管理起来
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        // 线程池调用执行到这里，再进入到真正的业务逻辑
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                // 线程池执行
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
`SerialExecutor`提供了任务的管理，实际的执行，是由线程池执行。`r.run()`才进到真正的业务逻辑中。即`FutureTask.run()`

## FutureTask.run
```java
    public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            // 这里拿出之前初始化的workerRunnable。
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);	// 把结果设置到outcome，然后再调用到done()接口。
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    protected void set(V v) {
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = v;
            U.putOrderedInt(this, STATE, NORMAL); // final state
            finishCompletion();
        }
    }
    
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (U.compareAndSwapObject(this, WAITERS, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();		// 回调到FutureTask的done实现。

        callable = null;        // to reduce footprint
    }
```
`FutureTask.run`函数，调用`workerRunnable.call`成功之后，继而设置`outcome`并且回调`done()`接口。

## WorkerRunnable.call
我们看下AyncTask初始化时，定义的动作
```java
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);		// AtomicBoolean对象，标记Worker被调用到了。这个标记，在FutureTask.done中，还会做判断
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);	// 调到doInBackground接口。
                    // 释放binder命令
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);		// 处理结果
                }
                return result;
            }
        };
```
在这里，通过线程池线程调用`call()`，进而调用`doInBackground(mParams)`，完成之后`postResult(result)`处理结果。

## AsyncTask.doInBackground(mParams)
```java
    /**
     * Override this method to perform a computation on a background thread. The
     * specified parameters are the parameters passed to {@link #execute}
     * by the caller of this task.
     *
     * This method can call {@link #publishProgress} to publish updates
     * on the UI thread.
     *
     * @param params The parameters of the task.
     *
     * @return A result, defined by the subclass of this task.
     *
     * @see #onPreExecute()
     * @see #onPostExecute
     * @see #publishProgress
     */
    @WorkerThread
    protected abstract Result doInBackground(Params... params);
```
这个抽象函数是关键，需要业务具体实现。我们看注释中描述，在这个方法中。可以调用`publishProgress()`，来更新UI线程。

## AsyncTask.publishProgress
```java
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```
如果没有`cancel`标记，则可以通过发消息通知`Handler`刷新UI界面(**前提是这个`Handler`是运行在主线程**)

## AsyncTask.postResult
```java
        private Result postResult(Result result) {
            @SuppressWarnings("unchecked")
            Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                    new AsyncTaskResult<Result>(this, result));
            message.sendToTarget();
            return result;
        }
```
把结果抛给`Handler`处理。

## AsyncTask.getHandler
```java
        private Handler getHandler() {
            return mHandler;
        }
        
        /**
        初始化时，赋予了具体对象
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
        **/
```
返回`mHandler`对象。如果之前初始化`AsyncTask`的时候没有传`Looper`，则默认为`InternalHandler`。如果有具体的`Looper`，则这消息将由具体业务处理。

## FutureTask.done
按照调用顺序，`postResult`发出消息之后，就会调用到`FutureTask.done`
```java
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
        
        /**
        public V get() throws InterruptedException, ExecutionException {
            int s = state;
            if (s <= COMPLETING)
                s = awaitDone(false, 0L);
            return report(s);
        }
        
        // 返回outcome，否则抛出Exception
        private V report(int s) throws ExecutionException {
            Object x = outcome;
            if (s == NORMAL)
                return (V)x;
            if (s >= CANCELLED)
                throw new CancellationException();
            throw new ExecutionException((Throwable)x);
        }
    
        private void postResultIfNotInvoked(Result result) {
            final boolean wasTaskInvoked = mTaskInvoked.get();	// worker.call的时候设置为true
            if (!wasTaskInvoked) {
                // 如果worker.call的过程出现Exception，这里确保能够结束任务。
                postResult(result);
            }
        }
        **/
```
这里的作用，就是确保`postResult`被调用到。从[FutureTask.run](##FutureTask.run)中，`mWorker.call`是有可能抛出`Exception`异常的，这里起了双保险的作用。


## InternalHandler.handleMessage
```java
    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    // 这个mTask是AsyncTask自身
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    // 调用onProgressUpdate接口，这个接口也会抽象函数，由具体业务实现
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
`Handler`处理两种消息，一个是`postResult`抛过来的，结束任务；是由`publishProgress`抛出，最后调用`onProgressUpdate`接口，这个接口也会抽象函数，由具体业务实现。

## AsyncTask.finish
```java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);		// 处理结束业务
        }
        mStatus = Status.FINISHED;
    }
```
至此，一个`AsyncTask`的任务就结束了。

# 注意事项
通过分析源码，我们发现有一个地方需要留意。
就是初始化`AsyncTask`的时候，注意传进去的`Looper`，是属于什么线程的。
如果这个线程不是UI线程，千万不要在`onProgressUpdate`方法中处理UI事务。