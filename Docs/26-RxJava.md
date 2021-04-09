#### RxJava

[建议根据此文章来学习](https://blog.csdn.net/carson_ho/category_7227390.html?spm=1001.2014.3001.5482)

[一](https://juejin.cn/post/6900870262062120967)

[二](https://www.jianshu.com/p/77cbb4a02c77)

```
RxJava是一个基于【事件流】、实现【异步操作】的库，应用的是通过一种扩展的观察者模式来实现的。
基于事件流的链式调用：
1、逻辑简洁
2、实现优雅
3、使用简单
即使随着程序逻辑的复杂性提高，依然能够保持简洁和优雅

有点：
1、基于链式调用，可以摆脱回调嵌套问题
2、更优雅的切换线程实现异步处理数据
3、配合操作符，处理事件流的代码更加简洁，逻辑更加清晰


主要包含以下几个角色：
Observable：俗称被订阅者，被订阅者是事件的来源，接收订阅者(Observer)的订阅，然后通过发射器(Emitter)发射数据给订阅者。

Observer：俗称订阅者，注册过程传给被订阅者，订阅者监听开始订阅，监听订阅过程中会把Disposable传给订阅者，然后在被订阅者中的发射器(Emitter)发射数据给订阅者(Observer)

Emitter：俗称发射器，在发射器中会接收下游的订阅者(Observer)，然后在发射器相应的方法把数据传给订阅者(Observer)。

Consumer：俗称消费器，这是RxJava2.0才出来的，在RxJava1.0中用Action来表示，消费器其实是Observer的一种变体，Observer的每一个方法都会对应一个Consumer，比如Observer的onNext、onError、onComplete、onSubscribe都会对应一个Consumer。

Disposable：是释放器，通常有两种方式会返回Disposable，一个是在Observer的onSubscribe方法回调回来，第二个是在subscribe订阅方法传consumer的时候会返回。

Subscribe：订阅

Event：事件流

1、创建被观察者 （Observable ）& 生产事件
2、创建观察者 （Observer ）并 定义响应事件的行为
3、通过订阅（Subscribe）连接观察者和被观察者

RxJava原理可总结为：
1、被观察者(Observable)通过订阅(Subscribe)按顺序发送事件给观察者(Observer)
2、观察者(Observer)按顺序接收事件(Event)和作出对应的相应动作

RxJava在android中切换到主线程也是通过Handler实现的

当使用了多个 subscribeOn() 的时候，只有第一个 subscribeOn() 起作用，作用于 subscribe()函数。每指定一次observeOn()线程则实现了一次线程的切换工作，在observeOn()之后的转换工作或者Subscriber都运行在observeOn()指定的线程中。


```

>##### RxJava 怎么通过被订阅者传给订阅者，过程是什么？
>
>1、订阅是从下游的Observer向上游的Observable发送订阅，然后在订阅的过程中，给下游的Observer发送订阅监听，并且给上游的被观察者添加订阅
>
>2、发送主要通过上游的被观察者通知发射器，然后发射器会发送给下游的Observer

> ##### Observer处理完onComplete后还能调用onNext吗？
>
> onComplete是用来控制不能发送数据的，也就是不能onNext了，包括onError也是不能再发送onNext数据了，该方法也是调用dispose方法

> ##### RxJava中map、flatMap的区别，还有其他哪些操作符？
>
> map是通过原始数据类型返回另外一种数据类型
>
> flatMap是通过原始数据类型返回另外一种被观察者
>
> concatMap和flatMap的功能是一样的， 将一个发射数据的Observable变换为多个Observables，然后将它们发射的数据放进一个单独的Observable。只不过最后合并ObservablesflatMap采用的merge，而concatMap采用的是连接(concat)。
>
> **concatMap是有序的，flatMap是无序的，concatMap最终输出的顺序与原序列保持一致，而flatMap则不一定，有可能出现交错。**
>
> 关于其他的操作符比如merge、concat、zip都是合并，interval是周期执行，timer是延迟发送数据。

> ##### Maybe、Observer、Single、Flowable、Completable几种观察者的区别，以及他们在什么场景用？
>
> Maybe：没有onNext方法，也就是说不能发多条数据，如果回调到`onSuccess`再不能发消息了，如果直接回调`onComplete`相当于没发数据，也就是说Maybe可能不发送数据，如果发送数据只会发送单条数据。
>
> Observer：能发送多条数据的，直到发送`onError`或`onComplete`才不会再发送数据了，当然它也是可以不发送数据的，直接发送`onError`或`onComplete`。
>
> Single：发送单条数据，单是它要么成功要么失败。
>
> Flowable：背压策略的一个被观察者
>
> Completable：不发生数据，只会发送成功或失败的事件

> ##### RxJava如何切换线程？
>
> `subscribeOn`指定被观察者的在哪个线程执行
>
> subscribeOn实际是创建了ObservableSubscribeOn的Observable，它的订阅方法里面创建了`SubscribeOnObserver`，通过线程池执行Runnable来达到上游Observable的订阅在子线程中执行，这就是为什么subscribeOn能控制observable在哪个线程中执行的原因。
>
> observeOn 指定观察者在哪个线程执行
>
> observeOn实际是创建了ObservableObserveOn的Observable，它的订阅方法里面创建了`ObserveOnObserver`，而`ObserveOnObserver`是实现了Runnable接口，把它包装成message给主线程的Handler发送一条消息，而`ObserveOnObserver`的run方法中会给下游的Observer发送数据。所以这就是observeOn能让observer在哪个线程中执行。

>##### RxJava的subscribeOn只有第一次生效？
>
>最开始调用的subscribeOn返回的observable会把后面执行的subscribeOn返回的observable给覆盖了，因此只有第一次生效

> ##### RxJava的observeOn多次调用哪个有效？
>
> 多次调用最后一个控制有效

> ##### 通过doOnSubscribe来监听线程发生切换

> ##### RxJava 1.0/2.0/3.0的区别？
>
> 2.0 vs 1.0：
>
> 1、添加了背压策略Flowable
>
> 2、添加Observer的变体consumer
>
> 3、ActionN和FuncN改名
>
> 4、Observable.OnSubscribe改为ObservableOnSubscribe
>
> 5、ObservableOnSubscribe中使用ObservableEmitter发射数据给Observer，1.0使用Subscribe发射数据
>
> 6、Subscription改为Disposable
>
> 3.0 vs 2.0：
>
> 1、提供Java 8 lambada 友好的API
>
> 2、删除一些方法

> ##### 如果不设置subscribeOn和observeOn，Observable和Observe在哪个线程执行？
>
> subscribeOn是控制上层Observable的线程执行，不设置即为发射数据的线程
>
> obsereOn是控制下游Observe的线程，如果不设置就跟上游Observable发射数据的线程保存一致。

> ##### 背压
>
> 指上游的Observable发射数据太快，下游的Observer接收数据跟不上导致的一种现象。
>
> **背压是发生在多线程中的问题，因为在单线程中，发送数据和接收数据都是在一个线程中，所以每次发送数据前得等到下游处理完数据才发送数据。**
>
> 1.0：
>
> 底层通过上游Observable发送的数据放到队列中，定义队列的容量是16，每次在往队列中放数据的时候会先获取下一个要放数据的索引，如果发现索引位置的数据不为空，则认为队列已经满了，那么满了就直接返回onError的信息。**通过设置上游发送过来的数据的接收队列容量来达到背压。**
>
> 2.0：
>
> MISSION：上游没有限流，在下游里面发现队列满了，给下游发送onError的信息，该信息是`io.reactivex.exceptions.MissingBackpressureException: Queue is full?!`。
>
> ERROR：在上游通过限流的形式给下游发送数据，在发现数据量到了128个的时候，会给下游发送onError的信息，该信息是`create: could not emit value due to lack of requests`。
>
> DROP：也是在上游通过限流的形式给下游发送数据，在发现数据量到了128的时候，会等下游处理完这128个数据，等到处理完了，继续处理梳理，所以在等的过程中会有数据丢失的问题。
>
> LATEST：虽然和DROP都是同样的丢数据，但是它两的做法是不一样的，LATEST通过只能放一个数据的容器来给下游发送数据，最开始丢数据基本是一样的，但是LATEST会保留最后一条数据，是因为最后处理数据的时候，容器里面还有一条数据。
>
> BUFFER：这个跟RxJava2.0普通发送数据是一样的，它不支持背压，上游发送多少数据，下游会接收多少数据，直到发生OOM。