#### ListView

```
https://blog.csdn.net/guolin_blog/article/details/44996879
https://blog.csdn.net/wangzhibo666/article/details/87370137

猜测：
	在ListView进行滑动的时候，如果RecycleBin中的mScrapViews里面没有缓存，则直接调用getView()重新创建，如果有缓存直接获取；滑出的布局会被RecycleBin缓存到mScrapView中，如果在滑动过程中只有回收，没有复用，则mScrapView会将滑出的View全部缓存
```

