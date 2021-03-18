#### Glide 

```
缓存策略
内存缓存：
	1、正在使用的图片缓存放到弱引用的一个map对象里面缓存
	2、不使用的图片缓存使用LRU算法实现LinkedHashMap里面
	
		加载图片时如果从缓存中获取到了图片会把图片从缓存中移除，然后加入到弱引用的缓存中，保护这些图片不会被LruCache算法回收掉
		EngineResource是用一个acquired变量用来记录图片被引用的次数，调用acquire()方法会让变量加1，调用release()方法会让变量减1，当acquired变量大于0的时候，说明图片正在使用中，也就应该放到activeResources弱引用缓存当中。而经过release()之后，如果acquired变量等于0了，说明图片已经不再被使用了，首先会将缓存图片从activeResources中移除，然后再将它put到LruResourceCache当中。这样也就实现了正在使用中的图片使用弱引用来进行缓存，不在使用中的图片使用LruCache来进行缓存的功能。
		
硬盘缓存：
	DiskCacheStrategy.NONE： 表示不缓存任何内容。
	DiskCacheStrategy.SOURCE： 表示只缓存原始图片。
	DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。
	DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。
	
	硬盘缓存的实现也是使用的LruCache算法

	
如果图片的配置信息发生了变化，Glide都会为其存储一份缓存，包括原图

如何监听生命周期变化


```

