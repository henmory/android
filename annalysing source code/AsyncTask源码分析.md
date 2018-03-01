# AsyncTask使用方法
 
  1.继承AsyncTask，实现内部抽象方法

  2.new AsyncTask子类对象

  3.调用子类的execute(params)，方法开启异步线程

# AsyncTask代码流程

  1.new AsyncTask子类对象的时候，需要在主线程中

  2.AsyncTask内部有个sHandler，它是一个静态的，这就意味着，sHandler是在主线程中声明，进而可以利用它把消息发送到主线程的消息队列中。

  3.sHandler的赋值在AsyncTask的构造函数中，因而在主线程中完成handler的创建以及相应的处理，其实在内部声明了一个mHandler指向sHandler，这就保证
  
  每一个AsyncTask子类都可以指向同一块代码（静态代码）＝＝＝》InternalHandler

  4.new AsyncTask的时候在其构造函数中，还为mWorker 和 mFuture赋值

  5.mWorker为一个 WorkerRunnable对象，mFuture为FutureTask对象

  6.WorkerRunnable类实现了Callable接口，因而它是我们的任务类，也就是它会运行在我们的线程池中，并调用doInBackground接口

  7.WorkerRunnable类还有一个params变量，存放传入任务的参数

  8.FutureTask类也继承自Runnable接口，进而它也可以作为认为类运行在线程中

  9.mFuture对象生成时，又把mWorker对象包装成FutureTask类，赋给mFuture

  10.进而任务类的调用过程会是mFuture.run()调用mWorker.call(),这两个只是返回值的区别

  11.调用execute(params)方法：executeOnExecutor(sDefaultExecutor, params);

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,Params... params) {

      onPreExecute(); //主线程调用

      mWorker.mParams = params;//赋值任务的参数

      exec.execute(mFuture); //告诉线程池去执行mFuture任务

    }
    
12.	exec==> sDefaultExecutor===>SerialExecutor（其实也是一个普通的类，运行在主线程中）

	private static class SerialExecutor implements Executor {

		final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();//任务的等待队列

		Runnable mActive;//线程池中是否有活跃的线程

		public synchronized void execute(final Runnable r) { //执行任务

			mTasks.offer(Runnable()); //等待队列添加任务

			if (mActive == null) {  //没有活跃的线程

			scheduleNext();//执行下一个任务

			}
		
		}

		protected synchronized void scheduleNext() {

			if ((mActive = mTasks.poll()) != null) {//取出一个将要执行的线程

			THREAD_POOL_EXECUTOR.execute(mActive); //线程池执行任务

			}

		}

	}
    
13.此时，主线程任务结束，进入到子线程中
	
	public void run() {  

		r.run(); //r 是 mFuture

		scheduleNext(); //执行等待队列中的下一个任务

	}
  
13. r.run()====>FutureTask.run()

	public void run() {

		result = c.call(); //执行mWorker的call
	
	}

14. WorkerRunable.call()

	public Result call() throws Exception {

		mTaskInvoked.set(true);//任务已经开始调起

		Result result = null;

		try {

			result = doInBackground(mParams);//开始执行费时任务

		} catch (Throwable tr) {

			mCancelled.set(true);

			throw tr;

		} finally {

			postResult(result); //结果执行完，返回值

		}

			return result;

	}

15.private Result postResult(Result result) {
    
    //获取主线程handler＝＝》sHandler发送消息和结果
    
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,new AsyncTaskResult<Result>(this, result));
    
    message.sendToTarget();
    
    return result;
  
  }

16.getHandler()＝＝＝》InternalHandler

private static class InternalHandler extends Handler {
    
    public void handleMessage(Message msg) {
        
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        
        switch (msg.what) {
            
            case MESSAGE_POST_RESULT:
                
                // There is only one result
                
                result.mTask.finish(result.mData[0]);//返回结果
                
                break;
            
            case MESSAGE_POST_PROGRESS:
                
                result.mTask.onProgressUpdate(result.mData);
                
                break;
        } 
    
    }

}

result.mTask 就是AsyncTask，调用其finish

17.private void finish(Result result) {
        
    if (isCancelled()) {  //如果过程中，用户调用cancel取消下载事件，那么用户可以调用onCancelled进行后续处理，由用户实现

        onCancelled(result);

    } else {//顺利执行完，调用onPostExecute由用户实现

        onPostExecute(result);

    }

    mStatus = Status.FINISHED;

}

18.如果用户需要在过程中，知道任务的进度，那么需要在doInBackground函数中，调用publishProgress(Progress... values)，value值用户可以自定义进

度（ 运行在子线程中）

19。自线程调用这个函数后，会发现，它会直接通过sHandler发送消息到主线程中

protected final void publishProgress(Progress... values) {
  
  if (!isCancelled()) {
      
      getHandler().obtainMessage(MESSAGE_POST_PROGRESS,new AsyncTaskResult<Progress>(this, values)).sendToTarget();
  
  }

}

20.进而我们可以在InternalHandler的handleMessage函数中的MESSAGE_POST_PROGRESS消息下，写主线程相关逻辑。

result.mTask.onProgressUpdate(result.mData); ＝＝＝》因而我们可以重写onProgressUpdate函数，做些有关界面的操作

# AsyncTask接口说明
	
  1.onPreExecute===>主线程提前做ui方面处理
  
  2.doInBackground(Params... params)===>子线程正在运行任务

  3.publishProgress(Progress... values)====>子线程时刻发布任务进度到子线程消息队列
  
  4.onProgressUpdate(Progress... values)=====>主线程接收到进度提醒消息，进行ui更新
  
  5.onPostExecute(Result result)====>主线程获取返回数据
	
  6.cancel====>主线程想主动取消任务执行
 
  7.onCancelled===>任务执行完后，主线程想做的事情

# 取消任务的流程＝＝》即用户主动调用cancel＝＝＝>主线程中

1.public final boolean cancel(boolean mayInterruptIfRunning) {
        
        mCancelled.set(true);
        
        return mFuture.cancel(mayInterruptIfRunning);//mTuture(runnable)取消
  }
  
2.public boolean cancel(boolean mayInterruptIfRunning) {
        
                    Thread t = runner;
                    
                    if (t != null)
                        
                        t.interrupt();//线程中断
               
            finishCompletion(); //完成后续事情
                
                return true;
    
    }
    
3.private void finishCompletion() {
    
    done();//主线程生存mFuture时复写了这个方法

}

4.mFuture = new FutureTask<Result>(mWorker) {
  
  protected void done() {

          postResultIfNotInvoked(get());
              
  };
  
5.private void postResultIfNotInvoked(Result result) {
    
    final boolean wasTaskInvoked = mTaskInvoked.get();
    
    if (!wasTaskInvoked) {
        
        postResult(result);//发送消息到主线程
    
    }
  
  }
  
又回到上面发送消息到主线程，MESSAGE_POST_RESULT

进而回到InternalHandler的handleMessage方法，最好又回到finish方法中，此时因为是取消线程进而会回调onCancel方法

#线程池配置

private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));[2,4]

private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;

private static final int KEEP_ALIVE_SECONDS = 30;

private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR;

static {

    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
    
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, 
            
            TimeUnit.SECONDS,sPoolWorkQueue, sThreadFactory);
    
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    
    THREAD_POOL_EXECUTOR = threadPoolExecutor;

}

1.静态的线程池，表示类加载到内存线程池就有了，也就是说，一个进程内所有使用AsyncTask的地方，共用一个线程池

2.由SerialExecutor的execute方法可知，它是一个synchronized函数，当我们用AsyncTask添加多个任务时，它是一个任务一个任务的放到等待队列中，

每放一个，判断线程池中是否有active的线程，如果有就不让线程池执行下一个任务，当线程池中的线程执行完这个任务后，会自动从等待队列中获取任务执行。这就说

明，线程池中，是只有一个线程在执行任务

3.因而如果想采用多线程并发处理任务，我们需要调用 public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor 

exec,Params... params) 这个接口，并给exec赋值为THREAD_POOL_EXECUTOR。这样每当我妈调用这个接口的时候，THREAD_POOL_EXECUTOR就给我们从线程

池创建一个线程执行任务，而不再需要SerialExecutor判断线程池中是否有active的线程，来了任务就开辟线程执行
