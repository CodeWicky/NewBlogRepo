---

title: TableView优化之加载图片的优化逻辑
layout: post
date: 2016-04-16 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- TableView优化 
- TableView图片加载
categories: 性能优化

---

系列文章：

- [TableView优化之高度缓存功能](http://www.jianshu.com/p/2b192257276f)

- [TableView优化之加载图片的优化逻辑](http://www.jianshu.com/p/328e503900d0)

- [TableView优化之快速滑动下的忽略加载](http://www.jianshu.com/p/0b020518de5e)
- - -
日常中，最常使用的空间非UITableView莫属了。
但是当TableView的cell中包含图片时，使用SDWebImage加载图片虽然是异步过程，但是仍然十分占用系统资源。
那么我们就要想一个办法去优化加载图片的逻辑。

此处，我只讲我自己的想法，或许也有更好的逻辑，还希望在下面留言指点我一下。

我的想法是TableView滚动的时候不去加载未加载过的图片，停止滚动后再从网络加载。已经加载过得图片，无论什么时候都加载该图片（因为SDWebImage会将加载过得图片缓存下来，再次加载的时候从缓存中取，这样就不用开辟线程下载图片了）。

<!-- more -->



![啊.png](http://upload-images.jianshu.io/upload_images/1835430-6fdc073348c239aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![屏幕快照 2016-04-16 下午9.34.14.png](http://upload-images.jianshu.io/upload_images/1835430-acb3f2d28681883e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上，就是我对TableView加载图片的优化逻辑。
转载标注
http://www.jianshu.com/p/328e503900d0
