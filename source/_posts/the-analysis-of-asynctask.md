---
title: AsyncTask 源码解析
date: 2017-05-30
tags:
  - Android
---

### AsyncTask 介绍

> AsyncTask 是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程并在主线程中更新 UI。从实现上来说，AsyncTask 封装了 Thread 和 Handler，通过 AsyncTask 可以更加方便地执行后台任务以及在主线程中访问 UI，当时 AsyncTask 并不适合特别耗时的后台任务，对于特别耗时的任务来说，建议使用线程池。

AsyncTask 现在估计使用并不多，但是在每个人刚开始学习 Android 开发的时候，相信都会学习并使用它。我们可以研究 AsyncTask 的源码，来加深对 Android 的理解，以达到触类旁通的能力。
<!--more-->
### AsyncTask 使用

AsyncTask 是一个抽象的泛型类，它提供了`Params`、`Progress`和`Result`这三个泛型参数，其中`Params`表示参数的类型，`Progress`表示后台任务执行进度的类型，`Result`则表示后台任务返回结果的类型。AsyncTask 这个类的声明如下所示：

```java
public abstract class AsyncTask<Params, Progress, Result>
```

AsyncTask 提供了 4 个核心方法，它们的含义如下：

1. `onPreExecute()`在主线程中执行，在异步任务执行之前，此方法会被调用，一般可以用于做一些准备工作。
2. `doInBackground(Param...params)`在线程池中执行，此方法用于执行异步任务，`parms`参数表示异步任务的输入参数。在此方法中可以通过`publishProgress`方法来更新任务的进度，`publishProgress`方法会调用`onProgressUpdate`方法。另外此方法需要返回计算结果给`onPostExecute`方法。
3. `onProcessUpdate(Progress...values)`在主线程中执行，当后台任务的执行进度发生改变时此方法会被调用。
4. `onPostExecute(Result result)`在主线程中执行，在异步任务执行之后，此方法会被调用，其中`result`参数是后台任务的返回值，即`doInBackground`的返回值。

下面提供一个典型的示例，如下所示：
```java
// 此示例摘自《Android 开发艺术探索》- 任玉刚
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
    @Override
    protected Long doInBackground(URL... params) {
        int count = params.length;
        long totalSize = 0;
        for (int i = 0; i < count; i++) {
            totalSize += Downloader.downloadFile(params[i]);
            publishProgress((int) ((i / (float) count) * 100));
            if (isCancelled()) {
                break;
            }
        }
        return totalSize;
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        super.onProgressUpdate(values);
        setProgressPercent(values[0]);
    }

    @Override
    protected void onPostExecute(Long aLong) {
        super.onPostExecute(aLong);
        showDialog("Downloader " + aLong + " bytes");
    }
}
```
当要执行上述下载任务时，可以通过如下方式来完成：
```java
new DownloadFilesTask().execute(url1, url2, url3);
```
在 DownloadFilesTask 中，`doInBackground`用来执行具体的下载任务并通过`publishProgress`方法来更新下载的进度，同时还要判断下载任务是否被外界取消了。当下载任务完成后，`doInBackground`会返回结果，即下载的总字节数。

### AsyncTask 源码
接下来，我们来看一下 AsyncTask 内部是如何实现的，注意，我这里以 Android 7.1 的源码为准，如果你看的是其他版本的源码，可能会有一些不同。

我们先看到构造函数吧：
```java
/**
* Creates a new asynchronous task. This constructor must be invoked on the UI thread.
 */
public AsyncTask() {
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
}
```
构造函数中，只是初始化了两个变量`mWorker`和`mFutrue`，并且`mWorker`作为`mFuture`的初始化参数。构造函数其他没有做什么实际性的操作，AsyncTask 总是从`execute()`开始的，我们看看`execute()`。
```java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
  return executeOnExecutor(sDefaultExecutor, params);
}
```
`execute`的执行是依靠`executeOnExecutor()`的。
```java
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

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
在函数开头会检查`mStatus`的状态，此时`mStatus`不能为`RUNNING`或`FINISHED`，这说明一个 AsyncTask 对象只能执行一次，即只能调用一次`execute`方法，否则会报运行时异常。
在上面的代码中，`sDefaultExecutor`实际上是一个串行的线程池，一个进程中所有的 AsyncTask 全部在这个串行的线程池中排队执行。在`executeOnExecutor`中，AsyncTask 的`onPreExecute`方法最先执行，然后线程池开始执行。下面分析线程池的执行过程。
```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
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
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```
结合 AsyncTask 的构造函数和`SerialExector`的实现可以分析 AsyncTask 排队执行的过程。首先系统会把 AsyncTask 的`Params`参数封装为`FutureTask`对象，接着这个`FutureTask`会交给`SerialExector`的`execute`方法去处理，`SerialExector`的`execute`会把`FutureTask`对象插入到任务队列`mTasks`中，如果此时`mActive == null`，即当前没有任务在执行，那么调用`SerialExector`的`scheduleNext()`来执行`mTasks`中的任务。当一个 AsyncTask 执行完成后，会继续调用`SerialExector`的`scheduleNext()`来执行下一个任务，直到`mTasks`中没有任务。从这一点可以看出，在默认情况下，AsyncTask 是串行执行的。

AsyncTask 中有两个线程池，`SerialExecutor`和`THREAD_POOL_EXECUTOR`和一个`Handler(InternalHandler)`，其中`SerialExecutor`用于任务的排队，而`THREAD_POOL_EXECUTOR`用于真正地执行任务，`InternalHandler`用于将执行环境从线程池切换到主线程。

在 AsyncTask 的构造方法中有如下这么一段代码，由于`FutureTask`的`run`会调用`mWorker`的`call`方法，因此`mWorker`的`call`方法会最终在线程池中执行。

```java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
		    mTaskInvoked.set(true);
			......
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
		    ......
}
```
在 `mWorker`的`call`中，首先将`mTaskInvoked`置为`true`，表示当前任务已经被调用过了，然后执行`doInBackground`，接着将其返回值传递给`postResult`方法，实现如下:
```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```
在上面的代码中，`postResult`通过`getHandler()`发送了一个`MESSAGE_POST_RESULT`的消息，`getHandler()`定义如下:
```java
private static InternalHandler sHandler;
private static Handler getHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler();
        }
        return sHandler;
    }
}

private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
可以发现，`sHandler`是一个静态的`Handler`对象，为了能将执行环境切换到主线程，这就要求`sHandler`这个对象必须在主线程中创建。**由于静态成员会在加载类的时候进行初始化，因此这就变相要求 AsyncTask 的类必须在主线程中加载，否则同一个进程中的 AsyncTask 都将无法正常工作。**`sHandler`收到`MESSAGE_POST_RESULT`这个消息后会调用 AsyncTask 的`finish`方法，如下所示:
```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

如果 AsyncTask 被取消执行了，那么就调用`onCanceled`方法，否则就会调用`onPostExecute`，可以看到`doInBackground`的返回结果会传递给`onPostExecute`方法。

### 参考资料
1. 任玉刚 - 《Android 开发艺术探索》