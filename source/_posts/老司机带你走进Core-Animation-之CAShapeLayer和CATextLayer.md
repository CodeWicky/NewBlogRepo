---

title: 老司机带你走进Core Animation 之CAShapeLayer和CATextLayer
layout: post
date: 2016-11-19 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CAAnimation 
- 核心动画
- CAShapeLayer
- CATextLayer
categories: 核心动画

---

![老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://upload-images.jianshu.io/upload_images/1835430-7a62b93ba150bff6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：

- [老司机带你走进Core Animation 之CAAnimation](http://www.jianshu.com/p/92a0661a21c6)
- [老司机带你走进Core Animation 之CADisplayLink](http://www.jianshu.com/p/434ec6911148)
- [老司机带你走进Core Animation 之几种动画的简单应用](http://www.jianshu.com/p/8e14616679ea)
- [老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://www.jianshu.com/p/3115050b7298)
- [老司机带你走进Core Animation 之图层的透视、渐变及复制](http://www.jianshu.com/p/dedc44fe8e35)
- [老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://www.jianshu.com/p/29cbc1744153)
- - -

呐，老司机之前说过会来讲CALayer的，当然不会食言啦，今天就讲一些CALayer相关的吧。

由于老司机这个想起来啥说啥的特点，CALayer与UIView的一些关系以及CALayer的一些重要属性，早在老司机CoreAnimation系列第一章里面就已经做了很系统的介绍。本着从来不凑字的原则（啧啧啧，违心），没看到的同学去[这里](http://www.jianshu.com/p/92a0661a21c6)补票吧。

不要怪老司机做事没有条理哦，毕竟当时也没想做成系列的，真的是有这么多读者才支持我一步一步写到这里。


![爱你们哟](http://upload-images.jianshu.io/upload_images/1835430-29aa5a4d1032747c.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

废话有点多，那今天要讲什么呢？就讲讲CALayer的两个子类，`CAShapeLayer`和`CATextLayer`吧。

<!-- more -->

- - -
### CAShapeLayer

其实在日常使用中，CALayer能满足需求的情况还是比较少的，（当然你用它来划线还是很好用的），原因就在于`CALayer并不能很方便`的生成除了矩形的`其他形状`。（其实老司机更愿意认为他是作为基类存在的，为所有子类提供公有属性及方法）由于作为基类的CALayer老司机已经介绍过了，所以接下来的两个子类老司机都会`只讲述其差异性`。然而`CAShapeLayer则`是作为一个强大无比的子类出现的，通过名字我们大概就可以猜到，`他可以画出各样的形状`。

- CAShapeLayer的优势

老生常谈了，肯定是性能啊（不提性能要如何装作一副很厉害的样子），他的`渲染都在GPU里面`，`不！占！内！存！`

- CAShapeLayer如何绘制出各种图形？

我们可以从头文件中观察到，CAShapeLayer有一个独有的属性，是`Path`。他是一个CGPathRef对象。我们知道，这就是个路径，没错，`CAShapeLayer就是根据这个路径绘制出各种形状的图形的`。

```
CAShapeLayer * circle = [CAShapeLayer layer];
    circle.bounds = CGRectMake(0, 0, 100, 100);
    circle.position = self.view.center;
    [self.view.layer addSublayer:circle];
    circle.path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, 100, 100)].CGPath;
    circle.strokeColor = [UIColor redColor].CGColor;
    circle.fillColor = [UIColor yellowColor].CGColor;
    circle.lineWidth = 10;
    circle.lineCap = @"round";
    circle.strokeEnd = 0.75;

```

![CAShapeLayer](http://upload-images.jianshu.io/upload_images/1835430-0eb13e22ef4f77d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
代码很简单，结合这张图片你就应该大概知道每个属性是做什么的。
挑几个讲一下吧：

 1.path
 
 可以看到，老司机这里用的是UIBezierPath生成一个path，然后取他的CGPath来获取路径的。他是什么呢？是`一层对CGPath的封装`，他更符合OC面向对象的语法风格。这都不是重点，老司机并不想讲怎么使用UIBezierPath。`重点是`这里有一个`初学者经常会犯的错误`，`同学们在绘制曲线的时候经常会以layer在父图层中的相对位置去绘制曲线，这是错的！！！应该以layer自身的坐标系划线。`别不当回事，你错的时候就知道咋回事了😈
 
 另外，如下图所示，`整个圆形`UIBezierPath其实是`分为多个子路径`绘制的，这个特性在CAKeyframeAnimation中会有特殊的应用（可以回顾一下[第一篇](http://www.jianshu.com/p/92a0661a21c6)）。
 

![子路径](http://upload-images.jianshu.io/upload_images/1835430-a6e13a035133e6aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
 2.strokeEnd
 
 是轮廓终点的属性，`取值范围[0,1]`。代表轮廓终点在整条路径的百分比处，相应的还有strokeStart属性。
 
 不过你应该思考的是：
 
 首先，`哪个是所谓的终点`？老司机可以告诉你答案，`靠上的那个点`是终点。那为什么0.75是在那个位置呢？请记住，`在iOS中，以x轴正方向（即水平向右）为0度，顺时针旋转一周为360度。`

其实说到这里CAShapeLayer的基本用法就结束了。


- 你这么说，意思是还有特殊用法咯？

说不上特殊用法，是两个`比较实用`但又偏门`的点`。

1.绘制空心图层


![绿油油的好护眼](http://upload-images.jianshu.io/upload_images/1835430-417f6bc522ded01e.gif?imageMogr2/auto-orient/strip)

大家看看上面这个简单的效果，看上去还可以是吧。

这个跟[第三篇](http://www.jianshu.com/p/8e14616679ea)里面那个系统更新样式采用的是两种画法，这个没有使用CADisplayLink做重绘。因为写这个demo时没有考虑到做暂停。

那这个怎么做呢？

把它分成两部分吧，一部分外面不变那部分，一部分中间变那部分。

这时候我们就要考虑`如何画出一个空心的图层`。

老司机先放一下空心图层的代码

```
CGRect rect = CGRectMake(0, 0, 100, 100);
    UIBezierPath * rectP = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:5];
    UIBezierPath * circleP = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(10, 10, 80, 80)];
    [rectP appendPath:circleP];
    CAShapeLayer * layer = [CAShapeLayer layer];
    layer.bounds = CGRectMake(0, 0, 100, 100);
    layer.position = self.view.center;
    layer.path = rectP.CGPath;
    layer.fillColor = [UIColor colorWithWhite:0 alpha:0.5].CGColor;
    layer.fillRule = kCAFillRuleEvenOdd;
    [self.view.layer addSublayer:layer];

```

可以看到，事实上老司机`叠加了两条路径`，一个圆角矩形一个圆形，然而这还并不够，因为`layer`还要知道`自己的填充判断规则`，就是重要的`fillRule`属性。

`这个属性是用来判断某一点是否在填充区域内的判断规则。`

他有两个枚举值，`kCAFillRuleNonZero`和`kCAFillRuleEvenOdd`。

这里介绍一下分别是如何判断的
     
-  kCAFillRuleNonZero
	从该点向任意方向画一条射线，若顺时针穿过该射线的条数与逆时针穿过该射线的条数不相等，则表示该点在区域内部，否则在外部。
	
- kCAFillRuleEvenOdd
	从该点向任意方向画一条射线，如果该射线穿过奇数条路径则该点在区域内部，否则在外部。
	
	
2.strokeEnd

为什么又说strokeEnd？你还说你不是凑字！

真不是，这次说他主要是想表达`这个属性是默认支持隐式动画的`。

隐式动画就是不用显示声明，系统默认为我们实现的动画。

网上99%的CAShapeLayer教程都是用这个属性做一个环形指示器，诶，`老司机就是不讲这个例子`，你们自己去想吧，无辜脸。

像下面这个图一样，不过他们都留了一个坑没说。我敢保证如果你只用strokeEnd和strokeStart两个属性交替配合，绝对实现不了这个效果。如果`不信邪你可以现在去试试`，啧啧啧。我会在文章的最后放出如何才能解决你们遇到的问题，别着急往下拉哦。（你要是没遇到问题，老司机再教你一个快捷键，`command + A`，`然后按delete键`可以快速整理代码）。



![这张图是我盗的](http://upload-images.jianshu.io/upload_images/1835430-efdce1871fc493f7.gif?imageMogr2/auto-orient/strip)

恩，这个strokeEnd的隐式动画讲完，上面老司机放的那个绿色背景进度图那个你也能做了，当给你们留的作业了自己去实现吧😊。


3.虚线

这个属性真的是`一直被忽略`，从未被使用。

![画了一条虚线](http://upload-images.jianshu.io/upload_images/1835430-48382570f6bb461b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码还是上面circle那段代码，在末尾添了三句话

	
	circle.lineWidth = 2;
    circle.lineDashPhase = 3;
    circle.lineDashPattern = @[@5,@5,@15,@5];
    
因为间距很小，原来的线宽显示不出来所以这里更改了线宽。

先看lineDashPattern这个属性。这个属性指的是实线与虚线长度交替的数组。注意`奇数位为实线`，`偶数位为虚线`，`单位像素`。系统会按照给定数组自动重复设置虚线。

lineDashPhase这个属性是告诉系统从多少开始计算这个距离。比如上图中第一段实现的距离明显小于5，其实他是2，因为我们从3开始计算，5 - 3就剩2了。

说到这里，CAShapeLayer的一些用法就真的介绍完毕了。
- - -

### CATextLayer

相比CAShapeLayer，可能CATextLayer的用途更加单一一些，他可以用来`展示文字`。

那个，等会再关浏览器，你先听我说完😭

我知道，有UILabel，你完全不需要使用这个。
但是存在必定是有他的意义的。正如UILabel是已经封装完成的，有一些我们想用的功能UILabel不一定有，比如下面这个：

![歌词Label](http://upload-images.jianshu.io/upload_images/1835430-4a92c932f13a83d1.gif?imageMogr2/auto-orient/strip)

当然这个效果用两个label叠加再用一个mask也可以实现，不过`两个label实在是不优雅`。所以`老司机决定用CATextLayer来实现这个效果`。

先来讲一下CATextLayer的基本使用方法吧。

他的几个属性都是见名知意，就是跟label相差无几的属性。最简单的你想让他显示文字的话直接给string属性赋值就好了。

不要太简单，哈哈哈😆

CATextLayer我们就讲到这里。
怎么可能，我当然会把这个的实现方式告诉大家啊~

先给大家看一个效果：


![这个效果你一定会](http://upload-images.jianshu.io/upload_images/1835430-bdb264c3fde947b7.gif?imageMogr2/auto-orient/strip)

这个效果你一定会吧，一个绿色的CALayer，，上面盖了一个红色的CAShapeLayer，strokeEnd从0到1，对吧。

那怎么让他`只显示字的区域呢`？记得老司机曾经讲个CALayer的一个属性叫做`mask属性`么？对咯，就是`以一个CATextLayer做红色的CALayer的mask，CATextLayer的字体设置有颜色，背景设置透明色，这样就只能显示出红色的CALayer的文字部分了`~把他封装在一个UIView里面我们的展示歌词的Label就完成了。大家可以去[这里下载这个Demo](https://github.com/CodeWicky/DWLyricLabel.git)哦~

恩，其实只要是显示文字，CATextLayer都可以完成，想要定制一些UILabel没有的功能，都可以考虑使用CATextLayer。

CATextLayer也算是讲完了~

- - -

回过头来说之前留的那个坑。问题就是当你`第一循环结束`后你想把你的`strokeEnd恢复成0`你却发现他是`以动画形式恢复`回去的不是你要的效果对吧？这就是因为他的隐式动画了。因为这时候我们不需要他的动画是吧？知道原因就好办了，我们可以通过
CATransaction显式的关闭他的动画，恢复成0，再打开动画，是不是就行了？哈哈哈，就是这么简单。

```
[CATransaction begin];
[CATransaction setDisableActions:YES];
self.layer.strokeEnd = 0;
[CATransaction setDisableActions:NO];
[CATransaction commit];

```

- - -

坑填完了，看截图你们也知道老司机是熬夜给你们码字的吧，是时候点波赞👍加波关注💗了~
