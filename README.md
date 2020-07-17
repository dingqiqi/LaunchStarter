提高启动速度，核心思想就是 减少主线程的耗时操作。启动过程中  可控住耗时的主线程 主要是Application的onCreate方法、Activity的onCreate、onStart、onResume方法。

**通常我们会在Application的onCreate方法中进行较多的初始化操作，例如第三方库初始化，那么这一过程是就需要重点关注。** 而Activity中主线程通常只是View的解析和刷新操作了。

减少主线程耗时的方法，又可细分为异步初始化、延迟初始化，即把 主线程任务 放到子线程执行 或 延后执行。 下面就先来看看异步初始化是如何实现的。

执行异步请求，通常是使用[**线程池**](https://juejin.im/post/5e26ac2ef265da3e4a5831cf)，例如：

```java
        Runnable initTask = new Runnable() {
            @Override
            public void run() {
                //init task
            }
        };

        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(threadCount);
        fixedThreadPool.execute(initTask);
```

但是通过线程池处理初始化任务的方式存在三个问题：
>- 代码不够优雅
假如我们有 100 个初始化任务，那像上面这样的代码就要写 100 遍，提交 100 次任务。
>- 无法限制在 onCreate 中完成
有的第三方库的初始化任务需要在 Application 的 onCreate 方法中执行完成，虽然可以用 CountDownLatch 实现等待，但是还是有点繁琐。
>- 无法实现存在依赖关系
有的初始化任务之间存在依赖关系，比如极光推送需要设备 ID，而 initDeviceId() 这个方法也是一个初始化任务。

那么解决方案是啥？**启动器**！

** LauncherStart，即启动器，是针对这三个问题的解决方案，对线程池的再封装，充分利用CPU多核，自动梳理任务顺序。**

使用方式：
- 引入依赖：
- 划分任务
- 理清关系
- 添加任务，执行启动

```java
public class MyApplication extends Application {

    private static final String TAG = "MyApplication";
    
    @Override
    public void onCreate() {
        super.onCreate();
        
        TaskDispatcher.init(getBaseContext());
        TaskDispatcher taskDispatcher = TaskDispatcher.createInstance();

        // task2依赖task1；
        // task3未完成时taskDispatcher.await()处需要等待；
        // test4在主线程执行
        //每个任务都耗时一秒
        Task1 task1 = new Task1();
        Task2 task2 = new Task2();
        Task3 task3 = new Task3();
        Task4 task4Main = new Task4();

        taskDispatcher.addTask(task1);
        taskDispatcher.addTask(task2);
        taskDispatcher.addTask(task3);
        taskDispatcher.addTask(task4Main);

        Log.i(TAG, "onCreate: taskDispatcher.start()");
        taskDispatcher.start();

        taskDispatcher.await();
        Log.i(TAG, "onCreate: end.");
    }

    private static class Task1 extends Task {
        @Override
        public void run() {
            Log.i(TAG, Thread.currentThread().getName()+" run start: task1");
            doTask();
            Log.i(TAG, Thread.currentThread().getName()+" run end: task1");
        }
    }

    private static class Task2 extends Task {
        @Override
        public void run() {
            Log.i(TAG, Thread.currentThread().getName()+" run start: task2");
            doTask();
            Log.i(TAG, Thread.currentThread().getName()+" run end: task2");
        }

        @Override
        public List<Class<? extends Task>> dependsOn() {
        	//依赖task1,等task1执行完再执行
            ArrayList<Class<? extends Task>> classes = new ArrayList<>();
            classes.add(Task1.class);
            return classes;
        }
    }

    private static class Task3 extends Task {
        @Override
        public void run() {
            Log.i(TAG, Thread.currentThread().getName()+" run start: task3");
            doTask();
            Log.i(TAG, Thread.currentThread().getName()+" run end: task3");
        }

        @Override
        public boolean needWait() {
        	//task3未完成时，在taskDispatcher.await()处需要等待。这里就是保证在onCreate结束前完成。
            return true;
        }
    }

    private static class Task4 extends MainTask {
        @Override
        public void run() {
            Log.i(TAG, Thread.currentThread().getName()+" run start: task4");
            //继承自MainTask，即保证在主线程执行
            doTask();
            Log.i(TAG, Thread.currentThread().getName()+" run end: task4");
        }
    }
    
    private static void doTask() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
2020-07-17 12:06:20.648 26324-26324/com.hfy.androidlearning I/MyApplication: onCreate: taskDispatcher.start()
2020-07-17 12:06:20.650 26324-26324/com.hfy.androidlearning I/MyApplication: main run start: task4
2020-07-17 12:06:20.651 26324-26427/com.hfy.androidlearning I/MyApplication: TaskDispatcherPool-1-Thread-1 run start: task1
2020-07-17 12:06:20.657 26324-26428/com.hfy.androidlearning I/MyApplication: TaskDispatcherPool-1-Thread-2 run start: task3

2020-07-17 12:06:21.689 26324-26324/com.hfy.androidlearning I/MyApplication: main run end: task4
2020-07-17 12:06:21.689 26324-26427/com.hfy.androidlearning I/MyApplication: TaskDispatcherPool-1-Thread-1 run end: task1
2020-07-17 12:06:21.690 26324-26429/com.hfy.androidlearning I/MyApplication: TaskDispatcherPool-1-Thread-3 run start: task2
2020-07-17 12:06:21.697 26324-26428/com.hfy.androidlearning I/MyApplication: TaskDispatcherPool-1-Thread-2 run end: task3
2020-07-17 12:06:21.697 26324-26324/com.hfy.androidlearning I/MyApplication: onCreate: end.

2020-07-17 12:06:22.729 26324-26429/com.hfy.androidlearning I/MyApplication: TaskDispatcherPool-1-Thread-3 run end: task2
```

