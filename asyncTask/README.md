# 客户端app线程池管理优化

&emsp;&emsp;目前一般客户端有许多业务需要有耗时后台操作，线程池的使用非常广泛。但是线程池用法是五花八门，要么自己创建和维护线程池，要么使用android系统asyncTask，线程池没有统一的管理和监控。

自己创建和维护线程池，存在几个问题：

* 开发成本高，重复劳动浪费人力。
* 线程池滥用，导致手机资源消耗。
* 代码分散，不利于监控。

使用系统asyncTask，存在几个问题：

* 系统代码不可控，存在系统bug不好修复。
* 不同android系统版本，asyncTask实现逻辑不一致，问题不好定位。常见crash如下：

04-17 11:22:52.009: E/AndroidRuntime(22792): FATAL EXCEPTION: main

04-17 11:22:52.009: E/AndroidRuntime(22792): java.util.concurrent.RejectedExecutionException: pool=128/128, queue=10/10

04-17 11:22:52.009: E/AndroidRuntime(22792): at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:1961)

04-17 11:22:52.009: E/AndroidRuntime(22792): at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:794)

04-17 11:22:52.009: E/AndroidRuntime(22792): at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1315)


为解决以上问题，对线程池的使用做了以下几点优化工作：

## 统一app线程池

* 对外提供统一的线程池工厂类。
![image](https://raw.githubusercontent.com/juefei/android-tech-album/master/asyncTask/globalThreadPool.png)

* 针对引入外部三方库，其中使用系统asyncTask，内置线程池不易更改，apk启动时进行hook。
![image](https://raw.githubusercontent.com/juefei/android-tech-album/master/asyncTask/asyncTaskHook.png)

* 针对后台耗时任务且与UI线程有交互的情形，封装统一的TMAsyncTask供业务方使用。无需关心线程池，只需关注自己的业务逻辑实现。后面将详述。
* 针对简单的任务，无需关心结果，封装统一的TExecutor供业务方使用，支持UI线程，非UI线程，延迟执行等功能。
![image](https://raw.githubusercontent.com/juefei/android-tech-album/master/asyncTask/simpleTask.png)

## AsyncTask功能增强   
&emsp;&emsp;基于使用系统asyncTask存在的问题，改进点如下：

 * 任务串行执行改为并发执行，显著提升任务执行效率.
 * 扩大待执行的tasks队列长度.
 * 优化异常处理策略：采取抛弃队列中最老一个task，执行刚加入的最新task。不直接抛出异常导致程序挂掉。
![image](https://raw.githubusercontent.com/juefei/android-tech-album/master/asyncTask/asyncTaskException.png)

 * 增加线程检测保护，构造函数和execute必须UI线程。
![image](https://raw.githubusercontent.com/juefei/android-tech-album/master/asyncTask/asyncTaskExecute.png)

 * 增加性能埋点，可在数据统计平台查看。主要植入四个监控点：
        
        onPreExecute UI线程准备工作
        doInBackground 非UI线程耗时逻辑
        onPostExecute UI执行结果
        rejectedExecution 执行异常信息
![image](https://raw.githubusercontent.com/juefei/android-tech-album/master/asyncTask/asyncTaskMonitor.png)
    
&emsp;&emsp;TMAsyncTask的使用跟系统原生AsyncTask保持一致，从而在基础框架层所做优化最大限度减少业务方迁移成本。
