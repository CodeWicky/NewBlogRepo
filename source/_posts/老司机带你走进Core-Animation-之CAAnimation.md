---

title: 老司机带你走进Core Animation 之CAAnimation
layout: post
date: 2016-09-06 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CAAnimation 
- 核心动画
categories: 核心动画

---

![老司机带你走进Core Animation 之CAAnimation](http://upload-images.jianshu.io/upload_images/1835430-77eacb0858d3a263.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：

- [老司机带你走进Core Animation 之CAAnimation](http://www.jianshu.com/p/92a0661a21c6)
- [老司机带你走进Core Animation 之CADisplayLink](http://www.jianshu.com/p/434ec6911148)
- [老司机带你走进Core Animation 之几种动画的简单应用](http://www.jianshu.com/p/8e14616679ea)
- [老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://www.jianshu.com/p/3115050b7298)
- [老司机带你走进Core Animation 之图层的透视、渐变及复制](http://www.jianshu.com/p/dedc44fe8e35)
- [老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://www.jianshu.com/p/29cbc1744153)
- - -
为什么时隔这么久我又回来了呢？
###回来圈粉。
开玩笑的，前段时间ipv6被拒啊，超级悲剧的，前后弄了好久，然后需求啊什么的又超多，所以写好的东西也没有时间整理。不过既然我现在回来了，那么这将是一个井喷的时节。（感觉给自己立下了什么死亡flag，啧啧啧）

先说什么呢？说一下CAAnimation相关的吧，因为最近也刚好在研究这些东西。不过看到末尾之前，请你相信我今天讲CAAnimation绝对是有阴谋的。
- - -
#老司机带你走进Core Animation

事实上，老司机今天讲的只是核心动画（Core Animation）中的一小部分，`动画那部分`=。=

开发中，有的时候我们可爱的UI妹子或者产品妹子会说（表示一下纯属个人YY）：技术哥哥，这里能不能duang的一下弹出来呢？这里能不能转个圈出现啊？这里能不能加一些炫酷的效果啊？


![Duang](http://upload-images.jianshu.io/upload_images/1835430-07cc02042b300306.gif?imageMogr2/auto-orient/strip)

作为`高冷而又无所不能`的技术，能说不么？
###不能！
然后你就硬着头皮去写动画吧。

好在呢，苹果已经想到**`程序员也是要撩妹`**的这个需求，所以为我们提供了CAAnimation这个好东西，当然还贴心的帮我们又进行了一层`封装`，那就是`UIView得animate相关`的方法了。

然而封装好的东西，灵活度当然会降低了。所以既然是基于`CAAnimation`的，那我们就去自己调用好了！

所以呢，今天我打算说说CAAnimaiton。

这篇博客中可能会涉及到什么呢？

- CALayer与UIView
- CAAnimation与UIView animate

<!-- more -->

- - -

## CALayer

为什么会先说CALayer呢？以为`CAAimation实际上操作的都是Layer层`。

所以不知道CALayer会很头疼的，知道的偷学去冲杯咖啡吧。

>The CALayer class manages image-based content and allows you to perform animations on that content. Layers are often used to provide the backing store for views but can also be used without a view to display content. A layer’s main job is to manage the visual content that you provide but the layer itself has visual attributes that can be set, such as a background color, border, and shadow. In addition to managing visual content, the layer also maintains information about the geometry of its content (such as its position, size, and transform) that is used to present that content onscreen. Modifying the properties of the layer is how you initiate animations on the layer’s content or geometry. A layer object encapsulates the duration and pacing of a layer and its animations by adopting the CAMediaTiming protocol, which defines the layer’s timing information.

>If the layer object was created by a view, the view typically assigns itself as the layer’s delegate automatically, and you should not change that relationship. For layers you create yourself, you can assign a delegate object and use that object to provide the contents of the layer dynamically and perform other tasks. A layer may also have a layout manager object (assigned to the layoutManager property) to manage the layout of subviews separately.

这是官方文档对CALayer的解释，以老司机的英语水平翻译成人话就是：

CALayer是用来`绘制、渲染`图形的。

### 那他与UIView又有什么区别和联系呢？

事实上所有的UIView及其子类图形的绘制。UIView下有一个layer属性，这个layer层就是真正绘制UIView所要展示的东西的人。然而之所以要将CALayer封装成UIView则是为了给其`添加事件响应`。

因为`CALayer`是继承与NSObject的，所以他是`不具备响应`的。而`UIView是继承与UIResponder`的，UIResponder本身`可以提供事件响应`，再用CALayer给他绘制一个图像，那一个具有响应事件的对象就诞生了，他就是我们的UIView。


当我们创建一个UIView对象时，系统自动为我们创建一个CALayer对象,这个对象就是UIView的layer属性。

### CALayer有具体有什么属性呢？

前面有说过，layer层负责绘制和渲染，那么图像的基本属性他是一定要有的。

这里呢，老司机还是着重介绍一个常用和重要的属性吧。

CALayer有`bounds，position，frame`的属性，他呢，其实就是UIView中bounds，center，frame的映射。事实上，我们在layer中一般只操作bounds和position两个属性。由于有着对应关系，所以这个理解起来就相对容易，我就不展开讲解了。

CALayer还有一个非常重要的属性`anchorPoint`，即锚点。说起来这个概念还是真难解释，锚点就好像是体心，一个物体抽象成一个点的对象。还记得物理中的质点么？我们操作属性，比如移动，旋转，缩放等等都是以锚点作为`参考点`的。他是一个`CGPoint类型`的对象，他的`x、y取值范围都是0~1`。数值就是从锚点到layer的左上角（类比UIView的origin）的 横向/纵向 距离占layer得 width/height 的`比例`。


![锚点](http://upload-images.jianshu.io/upload_images/1835430-b882c8aef33998c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`contents`，寄宿图。事实上这个属性我们操作的也很少，不过还是可以解释一下的。他可以看做是`当前图层的背景图片`。就是在当前图层的背景展示一张图片。也就是说你甚至可以用一个UIView展示一张图片了，就是这样。不过你别看他的类型是id，可不代表你给他传什么对象都可以。事实上你只能传`CGImageRef`类型的数据进去。他的id类型是为了配合OS X系统的。哦，你说为什么不能传UIIamge？是这样的，`CALayer是在QuartzCore框架下`的，他是可以跨平台的，也就是在iOS和OS X上都可以，所以他当然不能让你传入UIKit下的对象了。`CALayer的所有属性都不能传入UIKit的对象`。

`transform`，仿射变换啦，这个东西其实挺多能讲的，那套图像变换的原理挺有意思的，不过我矩阵真的不会算了=。=延展的东西数学好的同学们自己看吧，其实想用还是挺简单的。

`cornerRadius、masksToBounds`。这两个属性比较常用啦，切圆角时候我们用的属性。

`opacity`，类比UIView的alpha。

`shadowOpacity、shadowColor、shadowPath、shadowOffset、shadowRadius`，这几个阴影相关的属性。这里需要注意的是`shadowOpacity的默认值现在是0`。也就是说默认效果下你是看不到任何阴影效果的，想要实现阴影想过请先`更改shadowOpacity为一个非0得小于等于1的值`。还有一点就是由于shadow是图层外的延展效果，要想看到shadow效果请`设置masksToBounds属性为No`。

`borderColor、borderWidth`，这是边框相关的属性。borderWidth默认值是0。

`mask`，这也是个有趣又有用的属性，`遮罩属性`。可以以一个layerA作为另一个layerB的遮罩，即layerB.mask = layerA。那么作为遮罩的layerA中的透明区域将被过滤掉，`layerB只显示layerA的非透明区域`。借助这个属性可以实现很多神奇的效果，比如歌词、图像展开等等。


![mask](http://upload-images.jianshu.io/upload_images/1835430-d7eec1186a9a4349.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CALayer还有一个其他的属性，不过平时使用的也不多，老司机就不逐一介绍了，有兴趣的同学可以自行了解哈。

基于CALayer的种种属性，以及CALayer还有很多子类，有了他们，我们能看到各种各样的UI控件，那我们又如何创建动画呢？
- - -


## CAAnimation

### CAAnimation是什么呢？

>CAAnimation is an abstract animation class. It provides the basic support for the CAMediaTiming and CAAction protocols. To animate Core Animation layers or Scene Kit objects, create instances of the concrete subclasses CABasicAnimation, CAKeyframeAnimation, CAAnimationGroup, or CATransition.

官方对CAAnimation的描述也并不多，其实不过说的的确都很到位。
CAAniamtion是一个`抽象类`，遵循着CAMediaTiming和CAAciotn两个协议。通过他的`4个子类`，我们可以`实现很多绚丽的动画效果`。

- - -

CAAnimation下有几个属性，下面逐一介绍

`timingFunction`，这个呢，指定的是动画的节奏。所谓节奏是什么什么意思呢？就是动画执行的快慢交替。有如下几个可选项

- kCAMediaTimingFunctionLinear//线性节奏，就是匀速
- kCAMediaTimingFunctionEaseIn//淡入，缓慢加速进入，然后匀速
- kCAMediaTimingFunctionEaseOut//淡出，匀速，然后缓慢减速移除
- kCAMediaTimingFunctionEaseInEaseOut//淡入淡出，结合以上两者
- kCAMediaTimingFunctionDefault//默认效果

`delegate`，代理。设置代理以后就会执行代理方法啦。

- -(void)animationDidStart:(CAAnimation *)anim;

- -(void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag;

一个是动画开始的代理，一个是动画结束的代理。

值得注意的是，结束代理中，有一个flag。他的意思是如果动画正常播放完成的话，flag为YES。如果没有播放完成或者被移除则返回NO。

`removedOnCompletion`，是否在播放完成后移除。这是一个非常重要的属性，有的时候我们希望动画播放完成，但是保留最终的播放效果是，这个属性一定要改为NO，否则无效。

- - - 
遵循着CAMediaTiming协议，所以他自然也有duration、beginTime等一些属性。

`beginTime`,动画的开始时间，准确的说应该叫滞后时间。就是当你将动画加载layer中后滞后多长时间开始播放。

>/* The begin time of the object, in relation to its parent object, if
 * applicable. Defaults to 0. */
 
这个属性很重要，我想要着重讲一下。上面是beginTime属性的注释。从中我们能够看出，这个`beginTime是与父对象有关`的。

由于CALayer和CAAnimation都遵循CAMediaTiming协议，所以那他都可以作为我们animation对象的父对象。

如果你将animation添加在一个动画组（CAAnimationGroup）中，那么ok，你直接按照下面的写法就好了。


	animation.beginTime = beginTime;

但是，如果你直接添加在一个layer上，他的父对象是layer。这个layer的时间线很可能是一个过去的时间。即使你添加上beginTime得到的仍旧是一个过去的时间。那么这个时候我们的动画就会显示在动画结束时的状态。为了避免这种状况，我们要修正我们的时间。代码如下：

```
CFTimeInterval time = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
animation.beginTime = time + beginTime;
```

CACurrentMediaTime()是马赫时间，是全局时间，也就是设备距离上次开机的时间。

convertTime是获取本地时间。该方法有两个参数，第一个是时间，第二个是时间所对应的图层。当第二个参数为nil时，则返回的就是第一个参数的值。

这样我们就能得到一个绝对时间。

这里很重要，不过引申下去就远了，想了解的同学点[这里](http://blog.csdn.net/yaozhuoyu/article/details/9566511)。

`duration`，是动画的持续时间。简单的概念我就一笔带过了吧，同样的还有`repeatCount，autoreverses`。这些都是一些基本属性，基本看单词就知道用途了。

![duration](http://upload-images.jianshu.io/upload_images/1835430-62f9e4502e71bbfd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`speed`，这个我也要说一下，他表示的是`当前对象的时间流速`。简单来说，如果speed是2，duration是3，那么经过1.5秒，我的动画已经播放完成了。s = v * t，s就相当于已经播放的动画，也就是长度为3个单位的动画，v相当于每秒播放多少动画，即每秒播放两个单位的动画，所以1.5秒后动画其实就播放完成了。


![speed](http://upload-images.jianshu.io/upload_images/1835430-bc4299e260ddf310.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`timeOffset`,是表示我将从动画的`第几秒开始播放`。比如一个duration = 3的动画，timeOffset = 2，那么此时我的动画从第二秒开始播放，播放到第三秒动画结束的状态后，立即变为动画的初始状态，也就是第0秒的播放状态，继续播放之第二秒的状态，然后动画结束。形象的说就好像你绕操场跑圈，以前你都是`从起点开始跑跑一圈`；这回你`从一半开始跑`，但是`同样跑一圈`，过了起点就还要再跑半圈，就是这样。


![timeOffset](http://upload-images.jianshu.io/upload_images/1835430-d73090e071359ba4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

fillMode，是播放结束后的状态。他有四个枚举值

- kCAFillModeForwards//保持结束时状态
- kCAFillModeBackwards//保持开始时状态
- kCAFillModeBoth//保持两者，我没懂两者是什么概念，实际使用中`与kCAFillModeBackwards相同
`
- kCAFillModeRemoved//移除

哦，这个属性使用的时候要设置`removedOnCompletion = NO`，否则你是看不到效果的。其实除了保持结束状态有效果，其他跟没效果也没什么区别。。可能也是我没看出来。。

- - -
其实说实话，我也不想讲这么些属性，不过这些铺垫也不好讲。

下面又到了我们最喜欢的淘宝推荐环节。不对，是代码环节。

## CABasicAnimation

前文说过，CAAnimation是一个抽象类，想要实现动画效果呢，是要使用其四个子类的。

首先我们就介绍一下CABasicAnimation。

看名字就知道，这是一个多么基础的动画啊=。=

CABasicAnimation用来`创建基于两个状态的动画`，你只需要给出两个状态，一个初始状态一个终止状态，系统自动为你将中间的动画补全。


	UIView * redView = [[UIView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
    redView.backgroundColor = [UIColor redColor];
    [self.view addSubview:redView];
    CABasicAnimation * animaiton = [CABasicAnimation animationWithKeyPath:@"position"];
    animaiton.removedOnCompletion = NO;
    animaiton.fillMode = kCAFillModeForwards;
    animaiton.duration = 2;
    animaiton.toValue = [NSValue valueWithCGPoint:CGPointMake(400, 400)];
    [redView.layer addAnimation:animaiton forKey:nil];
    


![移动](http://upload-images.jianshu.io/upload_images/1835430-7312aedda0c07f65.gif?imageMogr2/auto-orient/strip)

这已经是最简单的一个动画效果了。（无视起点位置不对=。=视频录制时机总是晚）


下面老司机逐句讲解一下。

首先呢，我创建了一个animation对象。那个`keyPath`是什么呢？是我们要修改的属性名。那，我们要给谁做动画？老司机说过，CAAnimation都是对CALayer进行展示动画的。所以这属性当然是`CALayer的属性`了。至于那些属性改变支持动画效果呢？command + 左键点进CALayer，我们能看到CALayer的每个属性。看他的注释，是不是最后一句有一个`animatable`。对了，有这个的，`就是支持动画`的。


![支持动画的属性](http://upload-images.jianshu.io/upload_images/1835430-24721d8aad9561dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来的三句有了上面的介绍就应该很清楚了，分别是完成时不移除，完成时保持末尾状态，播放时长2秒。

再往下看，我`给animation的toValue赋值`。

这是在做什么呢？不知道，我们就点进去看看。

这个看到有`toValue，fromValue，byValue`。
注释还挺多，分开看。老司机之前说过，`CABasicAnimation是要指定动画的初始状态和终止状态的`。那这三个属性就是用来指定状态的。

- from就是指定初始状态
- toValue就是终止状态
- by就是状态的增量

这三个值呢，`不能全为空`，以为这样你就一个状态也没有指定。

`也不能全不为空`，因为这样你就指定了三个状态，系统也不知道选哪两个。

若果你`指定了一个状态`，那系统将自动`以当前状态作为另一个状态`。

若你`指定了两个状态`，则系统`已这两个状态作为始末状态`。

还有一点要说的是，看属性也看的出来，这个value你是要`传入一个对象`的，所以老司机将CGPoint装在了一个NSValue中。

最后一句就是将动画填入layer中了，注意这里老司机图省事就没有给key。实际开发中`建议开发者填写key且key不可重复`，因为如果涉及到检测动画播放进度或者是移除动画之类的都要填写正确的key才可以。

好的，我们的动画已经初具规模了，我们一点一点完善。

	CABasicAnimation * animaiton = [CABasicAnimation animationWithKeyPath:@"position"];
    animaiton.removedOnCompletion = NO;
    animaiton.fillMode = kCAFillModeForwards;
    animaiton.duration = 2;
    animaiton.beginTime = CACurrentMediaTime() + 1;
    animaiton.repeatCount = MAXFLOAT;
    animaiton.toValue = [NSValue valueWithCGPoint:CGPointMake(400, 400)];
    [redView.layer addAnimation:animaiton forKey:nil];


代码改成了这个样子，跑一下看看是什么样子。


![延时](http://upload-images.jianshu.io/upload_images/1835430-bab04c38cdbb7b08.gif?imageMogr2/auto-orient/strip)

我们看到我添加这延时语句，同时改变了repeatCount让他无限循环。（这里你要注意我延时语句的写法）

我们再稍微改一下

	CABasicAnimation * animaiton = [CABasicAnimation animationWithKeyPath:@"position"];
    animaiton.removedOnCompletion = NO;
    animaiton.fillMode = kCAFillModeForwards;
    animaiton.duration = 2;
    animaiton.toValue = [NSValue valueWithCGPoint:CGPointMake(400, 400)];
    [redView.layer addAnimation:animaiton forKey:nil];
    animaiton.beginTime = CACurrentMediaTime() + 1;
    animaiton.repeatCount = MAXFLOAT;
    
发现变化了么？老司机把重复和延时放在了添加动画之后，这个时候动画是什么样子的呢？


![调整语句顺序](http://upload-images.jianshu.io/upload_images/1835430-a86ec17b9a2c3a31.gif?imageMogr2/auto-orient/strip)

延时没有了，也不重复了，怎么回事？？？

因为当你调用addAnimation：forKey：这句时，其实系统是对你传入的`animation进行了一次copy`，然后`把copy的这份添加在图层上`。这时你再改animation当然是不能更改动画效果的了。

**这里同时也引出了一点，当你将animation加入layer层以后，你就无法再更改动画效果了。**

其他的属性都是字面意思，老司机就不一句一句讲了。

- - -
## CAKeyframeAnimation

刚才也忘了说了，其实CABasicAnimation和CAKeyframeAnimation都是继承自CAPropertyAnimation。看名字就知道是根据属性来展示动画的。

所以CAKeyframeAnimation也是根据属性展示动画的，他与CABasicAnimation的不同点在于他可以`指定多个状态`，不局限于始末状态，这样`你的动画将更加灵活。`


	CAKeyframeAnimation * animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.values = @[[NSValue valueWithCGPoint:CGPointMake(0, 0)],[NSValue valueWithCGPoint:CGPointMake(100, 100)],[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
    animation.duration = 2;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    [redView.layer addAnimation:animation forKey:@"keyframe"];



![keyframe](http://upload-images.jianshu.io/upload_images/1835430-bf5c0269b3108fe8.gif?imageMogr2/auto-orient/strip)

还是逐句讲一下。

我先生成一个CAKeyframeAnimation对象，指定要做动画的属性。
然后`给values属性给了一个数组`。这什么意思呢？`CABasicAnimation是指定两个状态，而我们的CAKeyframeAnimation则是指定多个状态`，动画也的确按照我的规划移动了。

其实这里你应该注意到一点，有没有发现我的红色方形最开始是看不到全部的？因为还记得`锚点`那个概念么？他的初始值是CGPointMake(0.5,0.5),`所有动画都是以锚点为参考点`，所以说你指定的状态都是相对于锚点来说的状态。当然如果有需要，你可以修改锚点来实现一些特殊的动画效果。

改一下代码

	CAKeyframeAnimation * animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.values = @[[NSValue valueWithCGPoint:CGPointMake(0, 0)],[NSValue valueWithCGPoint:CGPointMake(100, 100)],[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
    animation.duration = 2;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.calculationMode = kCAAnimationCubicPaced;
    [redView.layer addAnimation:animation forKey:@"keyframe"];
	

![calculationMode](http://upload-images.jianshu.io/upload_images/1835430-0b3020d9906c97d5.gif?imageMogr2/auto-orient/strip)

我改变了`calculationMode`这个属性，有没有发现转折点出动作平滑了许多？恩，这个属性是用来改变补间动画的计算模式的。这个属性`是CAKeyframeAnimation独有的属性`。

继续修改

	CAKeyframeAnimation * animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.values = @[[NSValue valueWithCGPoint:CGPointMake(0, 0)],[NSValue valueWithCGPoint:CGPointMake(100, 100)],[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
    animation.duration = 2;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.keyTimes = @[@0,@0.8,@1];
    [redView.layer addAnimation:animation forKey:@"keyframe"];

我去掉了calculationMode属性，添加了keyTimes属性，效果是这样的。


![keyTimes](http://upload-images.jianshu.io/upload_images/1835430-d64671e9c70a19f9.gif?imageMogr2/auto-orient/strip)

是不是发现第一段路径速度明显比第二段慢？是因为`keyTimes属性指定的是当前状态节点到初始状态节点的时间占动画总时长的比例`。若果`不设置keyTimes则匀速播放`。

Go on

	CAKeyframeAnimation * animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.values = @[[NSValue valueWithCGPoint:CGPointMake(0, 0)],[NSValue valueWithCGPoint:CGPointMake(100, 100)],[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
    animation.duration = 2;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.keyTimes = @[@0,@0.8,@1];
    animation.path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(100, 100, 100, 100)].CGPath;
    [redView.layer addAnimation:animation forKey:@"keyframe"];



![path](http://upload-images.jianshu.io/upload_images/1835430-1fbc0c1de0032f9e.gif?imageMogr2/auto-orient/strip)


这回机智如你应该发现几个问题，第一路径变了。

因为我们`指定了path属性`，所以这时`values属性将被忽略`，按照指定的path运动。（此处注意我给的是CGPathRef类型数据，为什么？还记得之前说的UIKit的事么？）

第二，即使以path为运动给你好像也不太对？

这是因为以path为运动轨迹后只是相当于用另一种方式填写了values属性。这是一个圆形的路径，他被分成了四个`子路径`。然而你在keyTimes中指定的时间状态只有第二个 状态到初始状态的时间比例为1，自然后面的动画就没有了。

改成下面的样子就好了。

	CAKeyframeAnimation * animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.values = @[[NSValue valueWithCGPoint:CGPointMake(0, 0)],[NSValue valueWithCGPoint:CGPointMake(100, 100)],[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
    animation.duration = 2;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.keyTimes = @[@0,@0.25,@0.5,@0.75,@1];
    animation.path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(100, 100, 100, 100)].CGPath;
    [redView.layer addAnimation:animation forKey:@"keyframe"];
    



![修改keyTimes](http://upload-images.jianshu.io/upload_images/1835430-6511471e8429dfda.gif?imageMogr2/auto-orient/strip)

这里呢，老司机想多嘴一句。当`使用path`这个属性时需要`注意这么一点`。就是当你使用的path是`两个Path拼接`到一起，比如一个波浪线你可以以两个半圆去模拟（当然你也可以使用二次贝塞尔曲线）。这时候其实你的动画是有`两个子路径的`。如果你`不设置calculationMode`属性的话，两个子路径之间`会有短暂的停顿`。所以这里`建议修改calculationMode为kCAAnimationPaced或者kCAAnimationCubicPaced`。这样动画将更加圆润。


事实上，CAKeyframeAnimation还有一个独有的属性，`rotationMode`。这个属性的`默认是一个空值nil`。有两种可选值，*kCAAnimationRotateAuto*或者* kCAAnimationRotateAutoReverse*。如果这个属性设置成以上两个值中的任意一个，当前layer都会`始终保持朝向运动方向`，也就是跟随运动方向`自动旋转`。

为什么特殊说一下这个属性呢？因为大家知道我们时常相对一个物体做旋转动画，然而我们又`不单纯的想以layer的中心坐旋转`，很多时候可能是layer的某个端点。这个时候你有三种选择：

- 更改锚点
- 更改你的layer层
- 结合移动和转动

`更改锚点`就是将锚点移至你想旋转的旋转中心。但是其实老司机`不建议你修改锚点`。因为锚点是一个layer层的参考点。当你修改锚点以后`将会影响layer的所有相关属性`以至于造成一些你所不希望的后果。

`更改layer层`就是`扩展当前layer区域`，以透明区域填补空白区域，强行让你所期望的端点成为旋转中心。这个方法实在`笨拙`，`不优雅`。

`结合移动和转动`就是其实`将以端点的转动拆分成锚点的转动及锚点的弧线运动`去模拟端点的转动。这个方法是三者中最优雅的了，最起码不是那些取巧的方法。

但是有了`rotationMode`这个属性，哈哈，这三个方法都不用了。你只需要一个`锚点的弧线动画`以及`rotationMode = kCAAnimationAutoRotate`就可以了，简单了很多吧。

- - -

以上就是基于CAPropertyAnimation的属性动画的基本使用方法，当然这里老司机一直使用移动动画做的示例，你可以`更改属性，去做其他的动画都是可以的`。另外温馨提示，`二级属性也可以哦`。

那你可能要说了，我使用过程中一般不可能只改变一个属性啊，我想两个一起改变怎么办啊？

我们的衣食父母苹果怎么会没想到呢？所以他为我们提供了一个类，CAAnimationGroup。

## CAAnimationGroup

看名字就看得出来是动画组了，他支持多个动画并发执行。

	CAKeyframeAnimation * animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    animation.values = @[[NSValue valueWithCGPoint:CGPointMake(0, 0)],[NSValue valueWithCGPoint:CGPointMake(100, 100)],[NSValue valueWithCGPoint:CGPointMake(0, 200)]];
    animation.duration = 2;
    animation.removedOnCompletion = NO;
    animation.fillMode = kCAFillModeForwards;
    animation.keyTimes = @[@0,@0.25,@0.5,@0.75,@1];
    animation.path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(100, 100, 100, 100)].CGPath;
    
    CABasicAnimation * animation2 = [CABasicAnimation animationWithKeyPath:@"cornerRadius"];
    animation2.toValue = @50;
    animation2.duration = 2;
    animation2.fillMode = kCAFillModeForwards;
    animation2.removedOnCompletion = NO;
    
    CAAnimationGroup * group = [CAAnimationGroup animation];
    group.duration = 2;
    group.fillMode = kCAFillModeForwards;
    group.removedOnCompletion = NO;
    group.animations = @[animation,animation2];
    
    [redView.layer addAnimation:group forKey:@"group"];


![CAAnimationGroup](http://upload-images.jianshu.io/upload_images/1835430-88b254bdbaa34902.gif?imageMogr2/auto-orient/strip)

其实用法差不多，不同是的首先我`不需要指定属性`了，因为CAAnimationGroup可以播放多个属性同时变化，所以不需要指定属性。

其次`不需要指定状态了`。因为CAAnimationGroup相当于多个CAAnimation动画并发执行的管理者，本身状态取决于其子动画的状态。

还有一个animations属性，这个属性用于指定他的子动画。这个属性有些两点，可以说一下。我们点进去可以看到，animations是`装有CAAnimation对象一个的数组`。这句话的重点就是CAAnimationGroup也继承与CAAnimation，所以`CAAnimationGroup是可以嵌套`的。


![animations](http://upload-images.jianshu.io/upload_images/1835430-f7569b4c0c423c81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

记得之前我说过beginTime很重要么？记得那个属性要加上矫正值么？想在你完全不用考虑那个问题了。遇到要改变beginTime的动画，`直接把他丢到CAAnimationGroup里面`去，然后时间你就直接加吧，也不用矫正了，哈哈哈哈。不过呢，你也不要高兴太早，这里`有一个坑`，就是`animations数组中你的所有CAAnimaiton对象请安beginTime进行升序排列`哦，否则也是会有问题的哟。

CAAnimationGroup是个好东西，简化了很多操作，同时还可以嵌套，属性还很少。
支持并发动画演示，屌屌的。
- - -

## CATransition

最后一个啦，转场动画。

	- (void)viewDidLoad {
	    [super viewDidLoad];
	    UIButton * bu = [UIButton buttonWithType:(UIButtonTypeSystem)];
	    [bu setBackgroundColor:[UIColor yellowColor]];
	    [bu setFrame:CGRectMake(200, 200, 100, 100)];
	    [self.view addSubview:bu];
	    [bu addTarget:self action:@selector(a:) forControlEvents:(UIControlEventTouchUpInside)];
	    self.view.backgroundColor = [UIColor grayColor];
	    UIImageView * redView = [[UIImageView alloc] initWithFrame:CGRectMake(100, 100, 100, 100)];
	    redView.backgroundColor = [UIColor redColor];
	    redView.image = [UIImage imageNamed:@"2.jpeg"];
	    [self.view addSubview:redView];
	    self.imgV = redView;    
	}
	-(void)a:(UIButton *)btn
	{
	    CATransition *animation = [CATransition animation];
	    animation.duration = 5;
	    animation.fillMode = kCAFillModeForwards;
	    animation.type = @"rippleEffect";
	    animation.subtype = kCATransitionFromTop;
	    [self.imgV.layer addAnimation:animation forKey:@"ripple"];
	    self.imgV.image = [UIImage imageNamed:@"3.jpg"];
    }


![transition](http://upload-images.jianshu.io/upload_images/1835430-34c04cf9d77613f7.gif?imageMogr2/auto-orient/strip)

看到了没有这个水波效果还是挺炫的。
这个动画有两点不同，一个是`type`,这个属性指定的是动画的主类型，有以下几种可选择的动画：

- kCATransitionFade
- kCATransitionMoveIn
- kCATransitionPush
- kCATransitionReveal

还有一些系统私有的动画但是可以安全使用,但是你只能用字符串

//@"cube" @"moveIn" @"reveal" @"fade"(default) @"pageCurl" @"pageUnCurl" @"suckEffect" @"rippleEffect" @"oglFlip"

还有一些绝对不能使用的动画我就不放出来了

另外一个属性是subType。
有四种选项：

- kCATransitionFromRight
- kCATransitionFromLeft
- kCATransitionFromTop
- kCATransitionFromBottom

见名知意，就是指动画的方向啦。


不过呢，这个转场动画不好的一点是需要跟转场代码写在一起，然而官方又没有给出转场到底是什么概念。

所以我目前测出来的能用的情况有，`imageView切换图片`，`控制器的push或modal`，`UIView对象调用exchangeSubviewAtIndex：WithIndex：方法`的时候可以出发转场动画，其他我真是没有测出来，有大神知道还望告知=。=十分感谢。

- - -

最后呢，鉴于有同学问到为什么做动画后，layer的frame还是没有发生改变，老司机翻了一下，之前真的忘了说，所以在这里补充一下。

除了图层树之外，系统在内部为我们的layer维持着三分copy。渲染树、呈现树、模型树。
图层树就是想UIView中的subview、superview一样的图层的树状逻辑。
他在CAAnimation中的各属性值是不随着动画而实时变化的。

渲染树是系统内部用来处理动画效果的一份copy。并且对我们是不公开的，我们无法调用。

呈现树是动画中每一个瞬时状态的各个属性值得那份copy，他是随着动画实时改变的。可以通过layer的presentationLayer进行访问。

模型树是系统内部用来表示动画目标状态的各属性值的copy，可以通过modelLayer进行访问。

- - -

以上呢，就是CAAnimation的基本使用方法，基于这个你还可以自己组合出很多绚丽的动画，老司机就不在在这卖弄了。

- - -
哦，老司机刚才说了，老司机写CoreAnimation的教程是有阴谋的，阴谋是什么呢？嘿嘿嘿~


![好多动画在一起](http://upload-images.jianshu.io/upload_images/1835430-e7d2fc246edd25c7.gif?imageMogr2/auto-orient/strip)

呐，这是一个老司机封装的DWAnimation，他可以轻松、优雅地帮助你生成上面的动画。(有些动画播放两次是因为demo中要展示某些属性的用处，不要在意)

比如这样：


![优雅么？](http://upload-images.jianshu.io/upload_images/1835430-04a7fd69f578b586.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面是gitHub地址：[DWAnimation](https://github.com/CodeWicky/DWAnimation)

这次老司机希望如果你喜欢，给我个star吧，哈哈哈。

- - -

## 参考资料

- [崔神写的关于CoreAnimation相关，个人认为全网最全！](http://www.cnblogs.com/kenshincui/p/3972100.html)

- [CAMediaTiming的详解](http://www.cocoachina.com/programmer/20131218/7569.html)

- [隐式动画和显式动画](http://www.cocoachina.com/ios/20150105/10812.html)

- - -
写在最后，有的朋友说过不喜欢我写博客的风格，呐这里我想解释一下。

首先呢，不是老司机做作，老司机平时就这样，`没事就好扯咸蛋`。。

再者呢，老司机写博客呢，也是纯粹`为了记录自己学习过的知识`，因为老司机自己也记不住。

三呢，老司机写博客呢，是`给一些刚入门的同学写的`，因为老司机一直都是在科普一些比较寻常的东西。既然是给入门的同学讲，老司机自然是能讲多细讲多细，知无不言言无不尽。但我要一直一板一眼的说，你看不到半小时保证就难以集中精力了。所以么，老司机是需要扯淡的。

四呢，由于老司机写博客都是想起来啥说啥，可能你们真要是做项目找资料的看到我的博客都嫌烦，没事的，`移步简书首页善用搜索功能`，我是给坐地铁无聊的小伙伴们打发时间写的。

最后，老司机就这样，
**你咬我啊。**


![你咬我啊](http://upload-images.jianshu.io/upload_images/1835430-6d94ce1bd36d5f61.gif?imageMogr2/auto-orient/strip)

- - -
