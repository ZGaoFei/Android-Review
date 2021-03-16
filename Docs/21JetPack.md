#### JetPack

##### Lifecycle

```     	
Lifecycle是怎样感知生命周期的？
在ComponentActivity中创建了ReportFragment放入ComponentActivity中，并初始化了LifecycleRegistry供getLifecycle调用然后注册LifecycleObserver，在ReportFragment中的各个生命周期都调用了dispatch(Lifecycle.Event event) 方法，传递了不同的Event的值。
相当于在Activity中创建了一个用于监听的Fragmet来回调各个生命周期方法

2.Lifecycle是如何处理生命周期的？
通过调用了((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);方法，也就是LifecycleRegistry 类来处理这些生命周期。

3.LifecycleObserver的方法是怎么回调是的呢？
LifecycleRegistry 的 handleLifecycleEvent方法，然后会通过层层调用最后通过反射到LifecycleObserver方法上的@OnLifecycleEvent(Lifecycle.Event.XXX)注解值，来调用对应的方法

4.为什么LifecycleObserver可以感知到Activity的生命周期
LifecycleRegistry调用handleLifecycleEvent方法时会传递Event类型，然后会通过层层调用，最后是通过反射获取注解的值，到LifecycleObserver方法上的@OnLifecycleEvent(Lifecycle.Event.XXX)注解上对应的Event的值，注意这个值是和Activity/Fragment的生命周期的一一对应的，所以就可以感知Activity、Fragment的生命周期了。

在Fragment中的处理方式是在Fragment初始化LifecycleRegistry供getLifecycle调用然后注册LifecycleObserver，在Fragment中的各个生命周期方法中都调用了dispatch(Lifecycle.Event event) 方法分发各个Event事件

实现LifecycleObserver接口的方式：
1、实现DefaultLifecycleObserver接口，然后重写里面生命周期方法；
2、直接实现LifecycleObserver接口，然后通过注解的方式来接收生命周期的变化；
对于这两种形式，Lifecycle.java文档中是建议使用第一种方式，因为文档中说明了，随着Java8成为主流，注解的方式会被弃用。

ObserverWithState内部持有LifecycleEventObserver和State，用于持有观察者的对象，便于存储
在addObserver()中会初始化ObserverWithState对象，然后将该对象存入mObserverMap，可以简单的把它理解成用来保存观察者的Map。

注意：在ON_CREATE/ON_START/ON_RESUME是在各个生命周期方法on***之后调用的，ON_PAUSE/ON_STOP/ON_DESTROY是在各个生命周期方法on***之前调用的
```

![](C:\Users\zhaogaofei\Desktop\get\Android-Review\images\lifecycle.png)