## Android项目组件化/模块化，用LiveData打造属于你组件通信框架EventBus

### 预备知识
1. Android 基础知识
2. LiveData的基本定义和用法
3. 开源消息框架EventBus的基本了解

### 看完本文能够学到什么
1. Android项目组件化/模块化
2. 组件通信框架Router/EventBus的探究
3. 用LiveData来实现EventBus

### 一、Android项目组件化/模块化
开发Android项目之初，项目业务比较单一，采用单Project的形式存在，随着业务线越来越多，团队的日益壮大，单项目的弊端是越来越明显。整个团队在单项目下开发，使得代码越来越臃肿，代码提交冲突多，业务线耦合严重。于是聪明的开发们提出了项目组件化/模块化。

![组件化图](/images/WX20191001-135632@2x.png)
现在基本上是这种模式了。我们现在主要关注的是moduleA、moduleB、moduleC、moduleD等module之间的通信。业务模块之间的关系是相互独立的，这样最大好处是业务代码解耦，开发者只需要管理自己的模块就OK了。但是很多场景是需要业务模块之间通信的，比如moduleA需要调用moduleB的一个方法，我们一开始module定义的接口放在业务支撑库common/base中，这样moduleA通过Router反射就能调用了。后面发现这样还是不方便，因为大家发现只要业务模块有重复逻辑或者暴露接口都往base里面放，这样慢慢的base也就变得臃肿了。

在此基础上我们改造了一下，不动base业务支撑库。将module分为业务模块和接口模块(extension)。

![组件通信](/images/WX20191001-142220@2x.png)

我们将接口定义在extension模块中，提供给自己和其他的module调用。这样我们可以不用再去改base了。又可以愉快的玩耍了。

当然根据某些场景来说，比如moduleA调用方法之后需要moduleB、moduleC、moduleD同时去更新UI，Router的能力就没法办到了，也就需要另外一种组件通信方式EventBus。EventBus这个框架很优秀，很多项目也会采用EventBus来弥补Router的不足。

但是个人认为EventBus带来的代码可读性太差，且约束力太差，开发者可以随意发即时消息，导致消息的错用滥用。这对团队开发来说效率反而是降低的。所以一般公司不会采用EventBus，都会自己研发消息框架。所以我们第一版消息框架出来了，关键代码如下：

```
private Map<String, List<WeakReference<MessageReceiver>>> messageReceivers = new ConcurrentHashMap();

//消息接受方
public interface MessageReceiver {
    void onReceive(@NonNull Message message);
}
```

我们为消息定了消息名，作为Map的key，而接收方是实现了MessageRecevier的对象，发送消息的时候根据消息名取到注册了MessageReceiver的List遍历，执行MessageReceiver.onReceive()，这个跟系统带的Observable与Observer是一样的，在单项目下也坚挺了一段时间。后面项目完全组件化了，暴露出来的问题越来越大。

1. **消息容易重名，因为各个module都是有自己的研发团队管理的，开发者对自己的模块是比较熟悉的，对于其他不负责的模块是两眼一抹黑，完全不了解了。且如果几个业务模块都需要收到消息，需要定一个统一的消息名，最好能够下沉到Base库，大家都能引用到，下沉这个事情我想大家内心是拒绝的。**
2. **消息没有绑定Activity/fragment的生命周期，可能造成内存泄漏。**
3. **消息约束力太差，容易造成消息错用滥用，特么好的没有学到，EventBus的毛病全学到了。**


为了解决以上痛点，借鉴了美团的消息框架，决定用LiveData来打造属于你的EventBus：

1. **消息名由moduleName + eventName组成，保证了唯一性，解决不同组件定义了重名消息的问题**

      采用两级HashMap的方式解决。第一级HashMap的构建以ModuleName作为Key，第二级HashMap作为Value；第二级HashMap以消息名称EventName作为Key，LiveData作为Value。查找的时候先用组件名称ModuleName在第一级HashMap中查找，如果找到则用消息名EventName在第二级HashName中查找。

2. **使用LiveData构建消息总线具有生命周期感知能力，使用者不需要调用反注册，相比EventBus和RxBus使用更为方便，并且没有内存泄漏风险。**

3. **消息总线框架增强以下约束：**

    * 只能订阅和发送在组件中预定义的消息。换句话说，使用者不能发送和订阅临时消息。
    * 消息的类型需要在定义的时候指定。
    * 定义消息的时候需要指定属于哪个组件。

定好思路接下来来实现了
```
@ModuleEvents(module = "moduleA")
public class DemoEvents {

    @ModuleType(Goods.class)
    public static final String EVENT1 = "event1";

    @ModuleType(User.class)
    public static final String EVENT2 = "event2";

    public static final String EVENT6 = "event6";
}
```

我们在extension模块定义消息，我们将根据ModuleEvents和ModuleType两个注解通过Processor生成EventDefineOfDemoEvents接口类，ModuleEvents注解是必须的，参数是module名，module名不是必须参数，如果不填，会通过gradle plugin获取moduleName。ModuleType参数传的消息实体。如果不填则实体是Object。

```
public interface EventDefineOfDemoEvents extends IEventsDefine {
  
  Observable<Goods> EVENT1();

  Observable<User> EVENT2();

  Observable<Object> EVENT6();
}

```

生成的类是extension模块暴露给其他模块的接口。

```
//发送消息
        EventBus
                .get()
                .of(EventDefineOfTimelineEvents.class)
                .EVENT1()
                .post(new Goods("23423432"));

 //订阅消息
        EventBus
                .get()
                .of(EventDefineOfTimelineEvents.class)
                .EVENT1()
                .observe(this, goods -> {
                    Log.e("Main", "goods1111 is " + goods);
                });

```
发送消息订阅消息通过动态代理根据moduleName和eventName去二级map中查找到对应的LiveData。就可以observe了。


