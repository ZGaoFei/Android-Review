#### Handler消息机制

###### Message

```
消息
里面包含了what、arg1、arg2、obj用来传递基本的数据，bundle用来传递复杂数据，但必须是实现了Parcelable接口
target是一个Handler对象，代表目标对象，为后面调用handler.handleMessage()回调到handler中处理数据
callback是一个Runnable，会回到Runnable.run()方法
next指向下一个Message，类似于链表

消息池的默认大小为50

obtain()：实际都是调用无参的obtain()，其他的都是进行赋值操作，obtain()方法中会获取息池中的第一个Message实体，并格式化消息体。
	重复的利用Message，避免重复创建消耗内存资源

recycle()：
	回收消息，将不再使用的消息加入消息池
```

Handler

```
处理机，主要是用于发送消息
匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露

构造函数
	public Handler(@Nullable Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    获取当前初始化Handler线程的Looper对象赋值给mLooper，然后获取Looper的消息队列
    1、当前线程在创建Handler的时候需要有Looper对象，不然会报错
    2、消息会被Handler发送到当前线程中Looper的MessageQueue里面
    3、Handler有且只能绑定一个线程的Looper
    
sendMessage()相关方法
	最终都会调用到sendMessageAtTime()方法中，这个方法会把msg.target=this，然后将msg放入MessageQueue中
	
dispatchMessage()
	public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
   处理消息回调的顺序依次是msg.callback.run()->callback.run()->handleMessage()
   
postAtFrontOfQueue()
	发送一个消息放入消息队列的头部

sendMessageAtFrontOfQueue()
	设置触发事件为0，从而使Message加入到消息队列的对头
```

##### MessageQueue

```
消息队列
	主要用来存放Message对象的，主要功能是想消息池中投递消息和取出消息
	
enqueueMessage()：投递消息
	msg.target不能为空，否则会报错
	
next()：取出消息
	nativePollOnce是阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。
	next()中，如果有消息屏障，则会优先处理异步消息，直到消息屏障被移除

quit():
	当safe =true时，只移除尚未触发的所有消息，对于正在触发的消息并不移除；
	当safe =flase时，移除所有的消息
	
enqueueMessage():
	添加一条消息到消息队列，按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。
	
removeMessages():移除消息
	采用了两个while循环，第一个循环是从队头开始，移除符合条件的消息，第二个循环是从头部移除完连续的满足条件的消息之后，再从队列后面继续查询是否有满足条件的消息需要被移除。
	
postSyncBarrier()：发送同步屏障
	该Message没有target，用于拦截同步消息，不会唤醒Looper
```

##### Looper 

```
永动机
	主要是不断循环执行，按分发机制将消息分发给目标处理
	
prepar(quitAllowed)：
	quitAllowed表示是否允许退出，为true表示允许
	每个线程只允许执行一次该方法，第二次执行时线程的ThreadLocal已经有值，会抛出异常
	主要主要是创建Looper对象，并保存到当前线程的ThreadLocal中
	调用Looper的私有构造方法，在构造方法里面创建了MessageQueue对象
	
loop()：
	loop()进入循环模式，不断重复下面的操作，直到没有消息时退出循环

	1、读取MessageQueue的下一条Message；
	2、把Message分发给相应的target，调用handler.dispatchMessage()
	3、再把分发后的Message回收到消息池，以便重复利用。
	
quit():
	调用MessageQueue.quit()
```

![](C:\Users\zhaogaofei\Desktop\get\Android-Review\images\handler_java.jpg)

**图解：**

- Handler通过sendMessage()发送Message到MessageQueue队列；
- Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target来处理；
- 经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理。
- 将Message加入MessageQueue时，处往管道写入字符，可以会唤醒loop线程；如果MessageQueue中没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作。

#### Handler Native层

[Handler native层](http://gityuan.com/2015/12/27/handler-message-native/)

![](C:\Users\zhaogaofei\Desktop\get\Android-Review\images\handler_arch.png)

图解：

- 红色虚线关系：Java层和Native层的MessageQueue通过JNI建立关联，彼此之间能相互调用，搞明白这个互调关系，也就搞明白了Java如何调用C++代码，C++代码又是如何调用Java代码。
- 蓝色虚线关系：Handler/Looper/Message这三大类Java层与Native层并没有任何的真正关联，只是分别在Java层和Native层的handler消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。
- WeakMessageHandler继承于MessageHandler类，NativeMessageQueue继承于MessageQueue类

另外，消息处理流程是先处理Native Message，再处理Native Request，最后处理Java Message。理解了该流程，也就明白有时上层消息很少，但响应时间却较长的真正原因。

##### ThreadLocal

```
线程本地存储区（Thread Local Storage，简称为TLS）每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。

set(T value)：将value存储到当前线程的TLS中

get()：获取当前线程的TLS区域的数据

1、每个线程都有一个ThreadLocalMap 类型的 threadLocals 属性。
2、ThreadLocalMap 类相当于一个Map，key 是 ThreadLocal 本身，value 就是我们的值。
3、当我们通过 threadLocal.set(new Integer(123))，我们就会在这个线程中的 threadLocals 属性中放入一个键值对，key 是 这个 threadLocal.set(new Integer(123))的threadlocal，value 就是值new Integer(123)。
4、当我们通过 threadlocal.get() 方法的时候，首先会根据这个线程得到这个线程的 threadLocals 属性，然后由于这个属性放的是键值对，我们就可以根据键 threadlocal 拿到值。 注意，这时候这个键 threadlocal 和 我们 set 方法的时候的那个键 threadlocal 是一样的，所以我们能够拿到相同的值。
5、ThreadLocalMap 的get/set/remove方法跟HashMap的内部实现都基本一样，通过 "key.threadLocalHashCode & (table.length - 1)" 运算式计算得到我们想要找的索引位置，如果该索引位置的键值对不是我们要找的，则通过nextIndex方法计算下一个索引位置，直到找到目标键值对或者为空。
6、hash冲突：在HashMap中相同索引位置的元素以链表形式保存在同一个索引位置；而在ThreadLocalMap中，没有使用链表的数据结构，而是将（当前的索引位置+1）对length取模的结果作为相同索引元素的位置：源码中的nextIndex方法，可以表达成如下公式：如果i为当前索引位置，则下一个索引位置 = (i + 1 < len) ? i + 1 : 0。

内存泄漏问题：
	由于ThreadLocalMap的key是ThreadLocal对象，且是弱引用，如果一个ThreadLocal没有外部强引用来引用它，在系统GC时，会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value。
	get、set、remove等方法中，都会对key为null的Entry进行清除，但是如果当前线程一直在运行，并且一直不执行get、set、remove方法，这些key为null的Entry的value就会一直存在一条强引用练：Thread Ref -> Thread -> ThreadLocalMap -> Entry -> value，导致这些key为null的Entry的value永远无法回收，造成内存泄漏。
	为了避免这种情况，我们可以在使用完ThreadLocal后，手动调用remove方法，以避免出现内存泄漏。
```

##### select/poll/epoll

```
select:
	select函数监控3类文件描述符，调用select函数后会阻塞，直到描述符fd准备就绪（有数据可读、可写、异常）或者超时，函数便返回。 当select函数返回后，可通过【遍历】描述符集合，找到就绪的描述符。
	缺点：
	文件描述符个数受限
	性能衰减严重：IO随着监控的描述符数量增长，其性能会线性下降;
poll:
	poll fd结构包含了要监视的event和发生的event，并且poll fd并没有最大数量限制。 和select函数一样，当poll函数返回后，可以通过【遍历】描述符集合，找到就绪的描述符。
	缺点：
	select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。同时连接的大量客户端在同一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其性能会线性下降。
epoll:
	epoll更加灵活，没有描述符数量限制。epoll使用一个文件描述符管理多个描述符，将用户空间的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。epoll机制是Linux最高效的I/O复用机制，在一处等待多个文件句柄的I/O事件。
	
对比：
	在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。)
	epoll优势：
		监视的描述符数量不受限制，所支持的fd上限是最大可以打开文件的数目
		IO性能不会随着监视fd的数量增长而下降。epoll不同于select和poll【轮询】的方式，而是通过每个fd定义的回调函数来实现的，只有就绪的fd才会执行回调函数。

如果没有大量的空闲或者死亡连接，epoll的效率并不会比select/poll高很多。但当遇到大量的空闲连接的场景下，epoll的效率大大高于select/poll
```

