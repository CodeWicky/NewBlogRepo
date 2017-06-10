---

title: 老司机带你走进Core Animation 之几种动画的简单应用
layout: post
date: 2016-10-17 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CAAnimation 
- 核心动画
- 动画
categories: 核心动画

---

![老司机带你走进Core Animation 之几种动画的简单应用](http://upload-images.jianshu.io/upload_images/1835430-1737a066b7319b0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：

- [老司机带你走进Core Animation 之CAAnimation](http://www.jianshu.com/p/92a0661a21c6)
- [老司机带你走进Core Animation 之CADisplayLink](http://www.jianshu.com/p/434ec6911148)
- [老司机带你走进Core Animation 之几种动画的简单应用](http://www.jianshu.com/p/8e14616679ea)
- [老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://www.jianshu.com/p/3115050b7298)
- [老司机带你走进Core Animation 之图层的透视、渐变及复制](http://www.jianshu.com/p/dedc44fe8e35)
- [老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://www.jianshu.com/p/29cbc1744153)
- - -

之所以要写这几种简单应用呢，是为了帮大家扩展一下思维，基于`CAAnimation`和`CADisplayLink`其实我们可以做到很多事情，不过我们都还是需要一个思路。有的时候可能，拿到一个效果，我们一眼就可以看出来，哦，使用核心动画就可以搞定，然而真正上手的时候就会发现，哦，没有想象的那么简单，为什么我达到的效果不对呢？一般情况下有两种可能，要么是思路不完整，要么是思路根本就不对。CAAnimation固然灵活，但要是使用方法不当的话，也会事倍功半。所以呢，今天老司机就针对以下几种情况来介绍截个动画的实现方式。（说这么多其实就是因为这段时间一直研究这个，的确也没研究别的，哈哈哈）

在这篇文章中你会看到以下一些内容：

- iOS中GIF动图的播放的实现方式

- iOS系统更新图标样式的实现方式

- 自定义水波样式的HUD的实现方式

接下来我们就对这三个内容逐一进行一下讲解。

<!-- more -->

- - -

### iOS中GIF动图的播放的实现方式

我们知道，在OC中展示静态图片我们是使用UIIamgeView的，然而UIImageView对GIF动画的展示却并不友好。我知道你们一定会说UIImageView不是有组动画么，老司机当然知道有这个api，老司机最开始也是用这个api，但是有的时候就会发现播放出的gif的节奏有可能会跟原图不太一样。这是为什么呢？原来这是因为GIF文件的原因。我们知道，GIF其实就是由很多帧静态图片组成后连续播放组成的。重点就在于其实`每一帧的时间是可以不一样`的。(虽然这样的情况的确占少数)其实这种情况还好一些，我们知道系统的组动画api是`不支持动画的暂停与恢复的`，而且当`图片很多的时候也有很大的崩溃几率`，这是我们所不希望看见的，那么我们就开始想自己写一个api让他完美解决以上问题？怎么做呢？

其实无非是UIImageView的图片不断切换，我自己加个定时器都可以。不过重要的还是一个思路。要想做到每一帧的时间可以不一样长，我相信用定时器很难实现吧。这个时候，我们是否可以换个思路，记得CAAnimation中可以指定每种状态时间的那个动画叫什么么还？对了，`CAKeyframeAnimation`。不记得了回头看看[这里的内容](http://www.jianshu.com/p/92a0661a21c6)。

既然我们使用CAKeyframeAnimation的话，动画的暂停与恢复我们自然可以控制，只要控制好内存也就可以解决崩溃问题，那么这就是我们的思路了。

有了思路就方便很多了，接下来我们只要去实现就好了。我记得以为伟大的程序猿曾经说过：
> 程序员最不怕的就是如何实现，不会就去网上找，最怕的就是没网和没思路。								------老司机


大体思路：

- 解析GIF文件，获取每一帧图片及对应时长

- 构建CAKeyframeAnimation动画，以layer的contents属性去展示图片

就这么简单的两步。

老司机这里就不上全部代码了，上一些核心的代码简单的展示一下如何实现：


![展示GIF](http://upload-images.jianshu.io/upload_images/1835430-d74c1c9bdeba5f3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


以上就是核心代码，其实每一句的目的都很简单，可能有些函数没用过大家自己简单查一下也就都明白了。然后老司机放一下自己写的UIImageView的GIF分类，大家可以直接拿去用，在这里：

[点我去下载](https://pan.baidu.com/s/1miApoko)

![效果图，中间是我点暂停了=。=](http://upload-images.jianshu.io/upload_images/1835430-871a34118f5e1d41.gif?imageMogr2/auto-orient/strip)

- - -

### iOS系统更新图标样式的实现方式

这个只得是什么呢？就是iOS中APP更新的时候在ICON上不是有一个更新的动画么？像下面这个样子：


![仿系统更新样式](http://upload-images.jianshu.io/upload_images/1835430-e4e8b0e5407ca525.gif?imageMogr2/auto-orient/strip)

这里我们就针对这个动画的实现方式进行一下探讨。

说实话拿到这个需求老司机第一反应是`CABasicAnimation`。因为我们知道这个`不规则的形状我们完全可以使用CAShapeLayer去绘制`（关于CoreAnimation中CALayer的个个子类老司机会在接下来的博客中逐一跟新，敬请期待）。`不同的path可以绘制出不同的形状，而且path属性又是animutable的`。貌似轻松地很。然而真正写了一下老司机发现自己还是太天真了，CABasicAnimation是基于始末状态的补间动画，然而老司机又根本拿不准他自动补充的计算方式，所以写出来的动画跟预期的总是有一定的差距，老司机的思路层一度陷入僵局。后来老司机`换了一个思路`，既然用不了补间动画，我就`一帧一帧画`吧。这里就用到了`CADisplayLink`（不熟悉的小伙伴来[这里补票](http://www.jianshu.com/p/434ec6911148)）。

大体思路：

- 找出能代表动画中每一种状态的path

- 使用CADisplayLink反复绘制

首先我们找一下这个路径要如何表示：

这里没有什么诀窍啊，高中数学啊，就是硬画啊，直接上代码


![路径计算](http://upload-images.jianshu.io/upload_images/1835430-2b42571913401c68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里属性和参数老司机解释一下`self.sRadius是中间非暂停状态下的扇形半径`。参数`percent是将要绘制的路径的角度百分比`，suspendR是大家能看到暂停状态下是从中心不断扩大一个圆的，`suspendR就是当前正要绘制的那个圆的半径`（注意并不是那个圆最终要变成的半径，而是当前的）。通过这三个变量老司机就能画出这个指示器各个状态的路径。（是不是很腻害，毕竟数学课代表，啧啧啧）

这里重点讲的是动画的绘制，calayer的绘制老司机会在接下来的博客里面慢慢介绍，本例中用到的`中空的layer使用`了两种绘制方式，`usesEvenOddFillRule`和`mask`两种，稍后放的demo里面会有具体代码，大家看一下就好。

接下来就是`使用CADisplayLink去一帧一帧绘制`了，这也是上一期讲过的内容了，老司机也不废话了，一切尽在demo吧：

[点我去下载](https://pan.baidu.com/s/1miApoko)

- - -

### 自定义水波样式的HUD的实现方式

闲的无聊写的一个效果，先看图：


![波浪指示器](http://upload-images.jianshu.io/upload_images/1835430-b41b9f8b7edb5e62.gif?imageMogr2/auto-orient/strip)

其实这个效果还挺有意思的，那么如何实现呢？

其实有了上面的铺垫你应该会马上反映出两种思路：CAAnimation动画或者一帧一帧绘制。

这里老司机说一句，本质上，`如果补间动画能完成效果的话，尽量使用CAAnimation`，不用一帧帧绘制，`代码量少了，cpu压力也小点`。但是一般情况写复杂的补间动画都画不出来，比如说这个。

我们还是分析一下需求，首先我们需要`一条浪线`，其次这条浪线`还要能涨高`。我们可以`通过改变正弦曲线的相位`来使波浪左右摆动起来，改变正弦曲线k值改变波浪的高度。（事实上老司机使用的是`三次贝塞尔曲线`模拟的正弦曲线，效果相似，只不过OC中没有正弦曲线的封装，想绘制正弦曲线的话会增加很多计算量）。

思路都在那，这个路径的绘制代码比较多，我就不截图了，也是在demo中大家看一下就好，一步一步思路都很清晰，还有老司机一向是注释详细的，你懂我~

[点我去下载](https://pan.baidu.com/s/1miApoko)

- - -

恩，这次主要是想给大家`提供一下思路的扩展`，毕竟学会了动画要会用才是正道。

另外，我还要推荐老司机出品的DWAnimation，帮你清爽的生成CALayer的属性动画，并且任意拼接，组合~

![优雅么](http://upload-images.jianshu.io/upload_images/1835430-1d727fcd1edc50a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

gitHub下载地址：[DWAnimation](https://github.com/CodeWicky/DWAnimation)
欢迎给我star哦~

恩，转载记得注明出处哦~

**如果你觉得本篇文章对你有帮助，麻烦你给我点个赞!**

**如果你就喜欢老司机这墨迹劲你再加个关注!**

**如果你爱上老司机了你也可以给我打赏!**

**但是就算你给我打赏我也只卖艺，不卖身。**


![不卖身](http://upload-images.jianshu.io/upload_images/1835430-b49f2dacb7fdccb4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
