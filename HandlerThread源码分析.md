**HandlerThread是什么？**
**HandlerThread是一个Android封装好的线程类，里面封装了Looper，无需关心Looper的创建和循环**

首先看一下HandlerThread的构造函数

```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;

```
**可以发现HandlerThread 继承Thread**

看一下run函数源码

```java
protected void onLooperPrepared() {
    }

@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```
**可以看到run函数里有Looper.prepare和Looper.loop()循环不断从MessageQueue里取Message，这是子线程用handler必须实现的过程****（不懂的看一下上一篇* [Android异步实现机制](http://blog.csdn.net/android_text/article/details/70228758)）

```java
synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
```
**这一端是为了异步线程Looper同步实现的操作**

```java
public Looper getLooper() {
   if (!isAlive()) {
       return null;
   }
   
   // If the thread has been started, wait until the looper has been created.
   synchronized (this) {
       while (isAlive() && mLooper == null) {
           try {
               wait();
           } catch (InterruptedException e) {
           }
       }
   }
   return mLooper;
}
```
**notifyAll()和wait实现同步机制**

结束当前线程的循环：quit()

```java
public boolean quit() {
    Looper looper = getLooper();
     if (looper != null) {
         looper.quit();
         return true;
     }
     return false;
 }
```
**所以在onDestory要调用一下quit()**

**=============================实战================================**

```java
HandlerThread handlerThread = new HandlerThread("my Thread") {
    @Override
    public void run() {
        super.run();

    }
};
handlerThread.start();
Handler mHandler = new Handler(handlerThread.getLooper()) {

    @Override
    public void handleMessage(Message msg) {
        //这是子线程
        switch (msg.what) {
            case 1:
                //TODO 执行耗时操作
                break;
            default:
                break;
        }
    }
};
mHandler.sendEmptyMessage(1);
```

```
handlerThread.quit();
```

**总结：**

 **1. HandlerThread是一个实现了Looper.prapare()和Looper.loop()的子线程。**
 
 **2. HandlerThead解决了不断的new Thread()来创建子线程做耗时操作，可以优化性能**
 
 **3. HandlerThread可以解决子线程和主线程交互，也可以解决子线程和子线程的交互**
 
 **4. 不用的时候一定要调用quit()。**

