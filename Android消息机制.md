### 预备知识
1. Android开发的基础知识
2. Handler、MessageQueue、Runnable、Looper的基本概念

### 看完本文可以达到什么程度
1. handler消息机制的原理
2. Android为什么需要handler
3. Handler为什么会有内存泄漏，如何解决

### 一、handler消息机制的原理

首先我们看下面一张图来回忆一下我们在开发的时候Handler,MessageQueue、Looper是怎么使用的。

![概念初探](/images/WX20190929-093139@2x.png)

* Handler可以处理Message，也可以将Message压入队列中。
* Message是存储消息Data的，MessageQueue队列中有Message，也有Runnable。
* Looper循环着去取Message来处理
* Runnable执行run方法，处理回调

再来看一组代码：
```
//处理消息
     Handler mMainHandler = new Handler(Looper.getMainLooper()) {

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
            }
        };


//发送消息

//post系列
    mMainHandler.post(runnable);
    mMainHandler.postDelayed(runnable, 1000);
    mMainHandler.postAtTime(runnable, 1000);
    mMainHandler.postAtFrontOfQueue(runnable);
//send系列
    mMainHandler.sendEmptyMessage(0);
    mMainHandler.sendMessageAtTime(message, 10000);
    mMainHandler.sendMessageDelayed(message, 1000);
    mMainHandler.sendMessageAtFrontOfQueue(message);
```

在使用Handler的时候API非常简单，对的，我们在设计机制或者框架都需要暴露出来的API越简单越好，所谓“大道至简”就是这么个道理。但当你使用Handler使用的炉火纯青时，是不是也会有种雾里看花的感觉，似懂非懂，总是搞不清楚Handler、MessageQueue、Looper之间的关系。
不知道大家注意到没，上面的代码有点不也一样的地方就是处理消息都是Message，但是发送消息post系列却能post一个Runnable。导致输入输出是不一样的。

**带着疑惑促使我去对源码分析，将我的结论画了张图，更加形象化的来解释它们之间的关系。**

![关系图](/images/WX20190929-101729@2x.png)

看图得出结论，一个线程对应着一个Looper对象，Looper对象自带并管理一个MessageQueue队列，MessageQueue对应着多个Message，Runnable会通过内部的转换getPostMessage(r)获取Message，而每个Message都绑定当前的Handler去处理。线程跟Handler并没有产生直接的关系，但是这样推到下来，线程和Handler是一对多的关系。

为了证明我的结论是正确的，还是需要从代码的角度来证明。还是一样从Handler.post(runnable)开始跟踪。post/send调用链路

```
Handler.post() -> Handler.getPostMessage() -> Handler.sendMessageDelayed() -> Handler.sendMessageAtTime() -> Handler.enqueueMessage() -> MessageQueue.enqueueMessage()
```
1. post系列和send系列都会都走Handler.sendMessageAtTime这个方法，殊途同归。
2. Handler.getPostMessage()中会通过Message.obtain()获得一个消息，runnable作为Callback。
3.在Handler.enqueueMessage()方法中msg.target = this， 将Handler与消息绑定。
4. MessageQueue.enqueueMessage 将Message压入Queue中。等着Looper循环取消息执行。

接下来要看看Looper这个类。Looper类中管理一个MessageQueue。

```
    final MessageQueue mQueue;
    //调用链路
    Looper.loop()->MessageQueue.next()->Message.target.dispatchMessage()->Handler.handleMessage()
```

1.Looper执行loop()方法，for(;;)循环去取mQueue中的消息，通过之前Message绑定的Handler分发消息，最后由Handler去处理消息。链接比较简单。小结一下链路是这样的Handler->Message->MessageQueue->Message->Handler。形成了循环圈。这个就比较有意思，google这样设计会不会让大家感到困惑呢？

至此我们搞清了Handler，MessageQueue、Looper之间的关系。前面已经讲了一个线程对应一个Looper对象。意思就是线程创建了Looper对象，比较有意思的就是创建的Looper对象分主线程和普通线程。这里探究一下主线程（ActivityThread）与Looer的关系。
```
ActivityThread.main()->Looper.prepareMainLooper()->Looper.loop()
```

1. Looper.prepareMainLooper创建了一个MessageQueue和Looper对象，并且将MessageQueue赋值给Looper的mQueue。普通线程也可以通过prepare()方法创建Looper对象，基本上是一样的，不同的是主线程是不允许退出的。

2. 调用loop，循环处理消息了。


分析完代码之后，画了张图来总结Handler、MessageQueue、Looper、Thread之间的关系。

![关系图](/images/WX20190930-225507@2x.png)

### Android为什么需要handler

以其说Android为什么需要Handler，不如问Android系统需要消息机制？我们知道app启动会启动一个主线程ActivityThread，展示界面测量、布局、绘制甚至于触摸事件、按键事件都会通过不同形式向主循环投递消息等待处理，所以我们监听页面是否渲染完可以用Looper.myQueue().addIdleHandler()的原因所在，也是应用“动”起来的根本所在。我想在操作系统层面来说都会有一个类似的消息机制。来保证系统的正常运行。

### Handler为什么会有内存泄漏，如何解决
看上面定义过的Handler。
```
     Handler mMainHandler = new Handler(Looper.getMainLooper()) {

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
            }
        };
```
Android studio 会直接警告上面写的代码存在内存泄漏。

This Handler class should be static or leaks might occur (anonymous android.os.Handler) less... (Ctrl+F1)
Since this Handler is declared as an inner class, it may prevent the outer class from being garbage collected. If the Handler is using a Looper or MessageQueue for a thread other than the main thread, then there is no issue. If the Handler is using the Looper or MessageQueue of the main thread, you need to fix your Handler declaration, as follows: Declare the Handler as a static class; In the outer class, instantiate a WeakReference to the outer class and pass this object to your Handler when you instantiate the Handler; Make all references to members of the outer class using the WeakReference object.

handler定义为内部类，可能会阻止GC。如果handler的Looper或MessageQueue 非主线程，那么没有问题。如果handler的Looper或MessageQueue 在主线程，那么需要按如下定义：定义handler为静态内部类，当你实例化handler的时候，传入一个外部类的弱引用，以便通过弱引用使用外部类的所有成员。