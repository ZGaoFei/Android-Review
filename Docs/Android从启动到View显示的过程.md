#### Android从启动到View显示的过程

##### Android的系统启动过程

```
1、长按开机键，引导芯片会从固化在ROM中的预设代码执行，然后加载引导程序BootLoader到RAM中执行，引导程序是Android系统运行前的一个小程序，主要作用是把系统拉起来运行
2、然后会启动内核进程swapper(pid=0)的进程和kthreadd(pid=2)进程，swapper进程主要是用于进程管理、内存管理，加载Display,Camera Driver，Binder Driver等相关工作；kthreadd会创建一些内核守护进程，是所有内核进程的鼻祖
3、当内核完成系统设置后，会在系统文件中解析【init.rc】文件，然后创建init进程，init进程会孵化出一些用户守护进程，还会启动servicemanager（binder服务管家）、bootanim(开机动画)等重要服务，init进程是所有用户进程的鼻祖
4、init进程会孵化出Zygote进程，Zygote是Android的第一个进程（虚拟机进程，创建JVM），Zygote是所有Java进程的父进程
5、fork Zygote进程会加载ZygoteInit类，注册Zygote Socket服务端套接字，加载虚拟机，提前加载类和资源
6、Zygote会fork出System Server进程，System Server进程负责管理整个Java Framework，包含AMS、PMS、WMS
7、Zygote fork出第一个APP进程是launch进程，用户可以看到的桌面APP，Zygote还会fork出其他APP进程，如电话、邮箱、浏览器等
```

##### Activity的创建过程

```
点击桌面的图标启动APP：
1、Launcher通知AMS要启动新的Activity，调用的是Launcher.startActivitySafely()，实质是调用Instrumentation.execStartActivity()，然后会通知AMS创建Activity
2、AMS会校验Activity是否正确，会检测在AndroidManifest中是否已经注册过，校验通过后，AMS会获取栈顶的Activity，通知所在的进程pause当前的Activity
3、Launcher进程pause掉当前的进程后，会通知AMS已经pause了Activity
4、AMS将Activity保证成ActivityStack，然后检测此Activity的进程是否已经存在
	a、如果存在，则交给当前进程LAUNCH_ACTIVITY，创建Activity
	b、不存在，通过Socket与Zygote进程进行通信，调用Process.start()，去fork一个新的APP进程
5、新的进程会导入ActivityThread类，并执行入口方法main()，执行main()是RuntimeInit通过反射的方式调用的
6、在main()中，会创建ActivityThread对象，然后调用ActivityThread.attach()
7、ActivityThread.attach()中会绑定ApplicationThread与AMS，用于AMS与ActivityThread进行通信
8、当绑定ApplicationThread成功之后，AMS会通知ActivityThread创建新的Activity，AMS会调用ApplicationThreadProxy，相当于调用ApplicationThread，然后ApplicationThread会通过handler将消息LAUNCH_ACTIVITY发送给ActivityThread
9、ActivityThread会执行handleLaunchActivity

调用startActivity等打开Activity
1、调用startActivity()最终都会调用startActivityForResult()
2、当前进程会通知AMS启动新的Activity
3、后面同上面的2-9
4、当新的Activity执行onResume()之后，AMS会通知栈顶的Activity执行onStop()
```

##### View的显示过程

```
1、handleLaunchActivity() 开始，会在handleLaunchActivity() 中初始化WindowManagerGlobal.initialize() ，实际是获取系统服务WindowsManagerService在客户端实现的对象，接着会调用performLaunchActivity()
2、performLaunchActivity()中，通过反射创建新的Activity对象，然后会调用activity.attach() 方法
3、activity.attach()：创建PhoneWindow对象，将WindowManager与PhoneWindow进行绑定
4、随后在performLaunchActivity()中会继续调用Instrumentation.callActivityOnCreate()方法，在callActivityOnCreate()方法中会调用activity.performCreate()，接着会执行onCreate()方法，在onCreate()方法中会调用setContentView()方法
5、setContentView()方法中是通过getWindow().setContentView()，在PhoneWindow.setContentView()中会installDecor()，在installDecor()中会初始化DecorView，PhoneWindow.setContentView()中使用LayoutInflater.inflate()的方式加载布局
6、接着会走到handleResumeActivity()->performResumeActivity()
7、performResumeActivity()方法中会调用activity.performResume()方法，在activity.performResume()方法中会先调用performRestart()方法，在performRestart()方法中按顺序走onRestart()->onStart()生命周期方法，接着会继续执行到mInstrumentation.callActivityOnResume(this)，调用activity.onResume()方法
8、handleResumeActivity()调用performResumeActivity()之后，接着将DecorView设置为INVISIBLE，然后通过windowManager.addView()与DecorView绑定到一起，在WindowManager.addView()实际是WindowManagerImpl.addView()方法中调用的是WindowManagerGlobal.addView()
9、WindowManagerGlobal.addView()方法中会创建ViewRootImpl对象，然后通过ViewRootImpl.setView()将DecorView进行绑定，然后在ViewRootImpl.setView()中会调用requestLayout()，之后会调用view.assignParent(this)，将DecorView的父View设置为ViewRootImpl
10、handleResumeActivity()之后会接着调用activity.makeVisible()将DecorView设置为VISIBLE
11、ViewRootImpl.requestLayout()->scheduleTraversals()，scheduleTraversals()会设置注册监听屏幕刷新信号的标志位（mTraversalScheduled）为true，然后发送一个同步消息屏障，然后Choreographer.postCallBack()将TraversalRunnable对象以CALLBACK_TRAVERSAL的消息格式放入消息队列中
12、Choreographer.postCallBack()会调用DisplayEventReceiver.scheduleVsync()->调用一个native方法向底层注册下一个屏幕刷新信号的监听
13、当接收到屏幕刷新信号后会回调到FrameDisplayEventReceiver.onVsync()方法，FrameDisplayEventReceiver是一个Runnable的实现，在onVsync()方法中通过handler将callback指向自己发送出去，然后会走到自己的run()方法中，这样做是为了切换到线程
14、FrameDisplayEventReceiver.run()会调用doFrame()方法
15、在doFrame()方法中，会执行doCallBack()，将保存在队列中的Runnable对象按照消息格式执行run()方法，这里就有CALLBACK_TRAVERSAL的消息格式的消息执行run()方法，回调到TraversalRunnable.run()，接着执行ViewRootImpl.doTraversal()方法
16、ViewRootImpl.doTraversal()方法中先将注册监听屏幕刷新信号的标志位（mTraversalScheduled）为false，然后移除同步消息屏障，然后会执行ViewRootImpl.performTraversals()方法
17、ViewRootImpl.performTraversals()会执行View的测量、布局、绘制方法
```

```
刷新View的流程：
刷新View会调用到View的invalidate()、requestLayout()方法，因此会向上进行遍历找到所有View的根布局，即ViewRootImpl，会执行invalidateChild()、invalidateChildInParent()、requestLayout()、requestChildFocus()等方法，这些方法都会调用ViewRootImpl的scheduleTraversals()方法，后面的步骤同上面的11-17
```

问题：

> 1、如何保证注册屏幕刷新信号的操作能及时执行？
>
> 如果是同一个线程则直接执行scheduleVsyncLocked()进行注册
>
> 如果不是同一个线程则通过mHandler.sendMessageAtFrontOfQueue(msg)发送一个最高优先级的消息来执行

>2、16.6ms
>
>屏幕以60HZ的刷新频率，即一秒钟刷新60次，每次刷新时间为1000 / 60 = 16.66...ms
>
> 屏幕会以固定的16.6ms时间刷新一次，这个值不固定，如果是90HZ的刷新率的屏幕，这个值就是1000 / 90 =11.111...
>
>屏幕会以固定的时间去缓存中读取数据显示到屏幕上

>3、如何保证同一个同步信号内不会重复注册监听，为什么要避免重复注册
>
>是通过标志位mTraversalScheduled来控制的，当注册监听后，会将标志位设置为true，在同步信号来之后，刷新屏幕的时候，会将标志位设置为false
>
>因为View的刷新是从上到下，即从ViewRootImpl开始，向下遍历刷新需要刷新的View，既然是遍历，就是在这一帧内有变化的View都会刷新，因此不需要多次注册。

> 4、同步消息屏障postSyncBarrier()的作用是什么
>
> 屏障消息的特性：
>
> Message分为3种：同步消息、异步消息、同步屏障消息
>
> - 屏障消息和普通消息的区别在于**屏障没有tartget**，普通消息有target是因为它需要将消息分发给对应的target，而屏障不需要被分发，它就是**用来挡住普通消息来保证异步消息优先处理的**。
>
> - **屏障和普通消息一样可以根据时间来插入到消息队列中的适当位置，并且只会挡住它后面的同步消息的分发**
>
> - postSyncBarrier()返回一个int类型的数值，通过这个数值可以撤销屏障即removeSyncBarrier()。
>
> - postSyncBarrier()是私有的，如果我们想调用它就得使用反射。插入普通消息会唤醒消息队列，但是插入屏障不会。
>
> 原理：
>
> 这个同步屏障的作用可以理解成拦截同步消息的执行，主线程的 Looper 会一直循环调用 MessageQueue 的 `next()` 来取出队头的 Message 执行，当 Message 执行完后再去取下一个。当 `next()` 方法在取 Message 时发现队头是一个同步屏障的消息时，就会去遍历整个队列，只寻找设置了异步标志的消息，如果有找到异步消息，那么就取出这个异步消息来执行，否则就让 `next()` 方法陷入阻塞状态。如果 `next()` 方法陷入阻塞状态，那么主线程此时就是处于空闲状态的，也就是没在干任何事。所以，如果队头是一个同步屏障的消息的话，那么在它后面的所有同步消息就都被拦截住了，直到这个同步屏障消息被移除出队列，否则主线程就一直不会去处理同步屏幕后面的同步消息。
>
> 同步消息是尽可能的宝座每一次接收到屏幕刷新信号第一时间执行

> 5、当View发起重绘请求时，会不会立刻进行刷新？
>
> 不会，当屏幕发起重绘请求的时候，会先向底层注册监听同步信号，当下一个同步信号到来后，才会发起计算操作，在计算完之后，会把数据放入缓存中，在屏幕下一帧中显示