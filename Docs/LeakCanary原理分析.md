#### LeakCanary

[LeakCanary原理分析](https://blog.csdn.net/chennai1101/article/details/103799424)

[LeakCanary原理解析](https://www.jianshu.com/p/261e70f3083f)

```
1、LeakCanary主要是通过Application的registerActivityLifecycleCallbacks方法监控每一个Activty的Destroy之后对象是否被收回

2、在Activity Destroy之后ActivityRefWatch的watch方法将被调用，watch方法会通过一个随机生成的key将这个弱引用关联到一个ReferenceQueue，然后调用ensureGone()；

当一个软引用/弱引用对象被垃圾回收后，Java虚拟机就会把这个引用加入到与之关联的引用队列中；
监测机制利用了Java的WeakReference和ReferenceQueue，通过将Activity包装到WeakReference中，被WeakReference包装过的Activity对象如果被回收，该WeakReference引用会被放到ReferenceQueue中，通过监测ReferenceQueue里面的内容就能检查到Activity是否能够被回收。

3、ActivityRefWatch的ensureGone（）方法中会先确认一次是否已经被回收，如果发现没有被回收，则主动GC一下，然后在次确认是否被回收，如果还是没有回收则判断为内存泄漏；

4、一旦确认是内存泄漏，则开始dump信息到hprof文件中，并调用heapdumpListener.analyze(heapDump)开始内存分析；

5、内存分析是在HeapAnalyzerService服务中进行的，属于一个单独的进程；

6、HeapAnalyzerService的runAnalysis中创建HeapAnalyzer对象并调用它的一个核心方法checkForLeak（）；

7、HeapAnalyzer的checkForLeak（）会先解析hprof文件并生成快照文件，然后对快照中的泄漏对象进行去重，去重后根据第2步中的key去获取泄漏对象，如果对象为空则说明对象已经被回收，如果不为空则通过findLeakTrace（）方法计算出最短GC路径，并显示到DisplayLeakActivity页面，提醒开发者存在内存泄漏；
```

