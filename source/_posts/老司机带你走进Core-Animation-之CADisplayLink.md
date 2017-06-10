---

title: 老司机带你走进Core Animation 之CADisplayLink
layout: post
date: 2016-09-21 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CAAnimation 
- 核心动画
- 计时器
categories: 核心动画

---



![老司机带你走进Core Animation 之CADisplayLink](http://upload-images.jianshu.io/upload_images/1835430-107159f2337784d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：

- [老司机带你走进Core Animation 之CAAnimation](http://www.jianshu.com/p/92a0661a21c6)
- [老司机带你走进Core Animation 之CADisplayLink](http://www.jianshu.com/p/434ec6911148)
- [老司机带你走进Core Animation 之几种动画的简单应用](http://www.jianshu.com/p/8e14616679ea)
- [老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://www.jianshu.com/p/3115050b7298)
- [老司机带你走进Core Animation 之图层的透视、渐变及复制](http://www.jianshu.com/p/dedc44fe8e35)
- [老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://www.jianshu.com/p/29cbc1744153)
- - -

今天说点啥呢？上次老司机说过，带你走进CoreAnimation，那今天就趁热打铁，继续讲讲核心动画相关的东西吧。那今天要讲的就是CADisplayLink。

这篇文章会涉及到什么呢？

- CADisplayLink的基本使用方法
- OC中的三种定时器：CADisplayLink、NSTimer、GCD
- runloop浅析

<!-- more -->

- - -
### CADisplayLink

点进CADisplayLink的头文件我们能看到，其实他的方法并不多，而且他的功能很单一，就是作为一个`定时器`的存在。

不过既然苹果专门提供了这么一个类，就一定是有他的存在意义的。他的优势就在于他的执行频率是`根据设备屏幕的刷新频率来计算`的。换句话讲，他也是`时间间隔最准确`的定时器。

还是在使用中介绍吧。

```
- (void)viewDidLoad {
    [super viewDidLoad];        
    self.view.backgroundColor = [UIColor grayColor];
    ///target selector 模式初始化一个实例
    self.timerInC = [CADisplayLink displayLinkWithTarget:self selector:@selector(changeImg)];
    ///暂停
    self.timerInC.paused = YES;
    ///selector触发间隔
    self.timerInC.frameInterval = 2;
    
    self.imgV = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 200, 200)];
    self.imgV.contentMode = UIViewContentModeScaleAspectFill;
    self.imgV.center = self.view.center;
    [self.view addSubview:self.imgV];
    
    ///加入一个runLoop
    [self.timerInC addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
    
    UIButton * button = [UIButton buttonWithType:(UIButtonTypeSystem)];
    [button setFrame:CGRectMake(0, 0, 100, 30)];
    button.center = CGPointMake(self.view.center.x, self.view.center.y + 200);
    [self.view addSubview:button];
    [button setTitle:@"开始播放" forState:(UIControlStateNormal)];
    [button setBackgroundColor:[UIColor whiteColor]];
    [button addTarget:self action:@selector(gifAction) forControlEvents:(UIControlEventTouchUpInside)];
}
-(void)changeImg
{
    self.currentIndex ++;
    if (self.currentIndex > 75) {
        self.currentIndex = 1;
    }
    self.imgV.image = [UIImage imageNamed:[NSString stringWithFormat:@"%ld.jpg",self.currentIndex]];
}

-(void)gifAction
{
    self.timerInC.paused = !self.timerInC.paused;
}

```


![CADisplayTimer](http://upload-images.jianshu.io/upload_images/1835430-8a5d35d64ec667cf.gif?imageMogr2/auto-orient/strip)

我们可以从头文件中看到，苹果只提供了一个生成实例的接口。

>+(CADisplayLink *)displayLinkWithTarget:(id)target selector:(SEL)sel;

通过这个方法，可以以target/selector模式`生成一个绑定了触发事件的实例`。参数target、selector可以类比button，我就不做具体讲解了。

然而你只生成一个实例你的事件是`不会被触发`的，这是因为你没有`把他加入到runloop`当中。

>-(void)addToRunLoop:(NSRunLoop \*)runloop forMode:(NSString \*)mode;

你可以调用这个方法将实例加入到一个选定的runloop中，这时我们的事件就能被触发了。

>-(void)removeFromRunLoop:(NSRunLoop \*)runloop forMode:(NSString \*)mode;

有添加当然会有移除，当你要从某个runloop中移除当前实例的时候你可以调用上面的方法。

类比NSTimer，CADisplayLink也有一个计时器销毁的方法：

>-(void)invalidate;

调用这个方法，会`从所有runLoop中移除当前实例`，这个方法可以用于不需要计时器后对他进行释放前的操作。

好吧，CADisplayLink就这四个方法。以及四个属性：

- `timestamp`，获取上一次selector被执行的时间戳。这个属性是一个`只读属性`，而且你要记住的是只有`当selector被执行过一次之后`这个值才会被取到有效值。这个属性同上是用来比较当前图层时间与上一次selector执行时间只差，从而来计`算本次UI应该发生的改变的进度`（例如视图做移动效果）。

- `duration`，获取当前设备的屏幕刷新时间间隔。同timestamp一样，他也是个只读属性，并且也需要selector触发一次才可以取值。值的一提的是，当前iOS设备的刷新频率都是60HZ。也就是说`每16.7ms刷新一次`。作用也与timestamp相同，都可以用于辅助计算。不过需要说明的一点是，如果`CPU过于繁忙`，duration的值是会`浮动的`。

- `paused`，看名字就能看出来，是控制计时器暂停与恢复的属性。设置为YES的时候会暂停事件的触发。

- `frameInterval`，事件触发间隔。是指两次selector触发之间间隔几次屏幕刷新，`默认值为1`，也就是说屏幕`每刷新一次，执行一次`selector，这个也可以间接用来`控制动画速度`。

两次selector触发的时间间隔是time = frameInterVal * duration。必须注意的是，`selector执行所需要的时间一定要小于其触发间隔，否则会造成掉帧情况`。

总体来说，CADisplayLink的使用还是比较简单的。

- - -
### 三种定时器的优势与劣势
#### CADisplayLink
基本用法上文刚刚介绍过。

优势：依托于设备屏幕刷新频率触发事件，所以其触发时间上是最准确的。也是`最适合做UI不断刷新的事件`，过渡相对流畅，无卡顿感。

缺点：

- 由于依托于屏幕刷新频率，若果CPU不堪重负而影响了屏幕刷新，那么我们的触发事件也会受到相应影响。
- selector触发的时间间隔只能是duration的整倍数。
- selector事件如果大于其触发间隔就会造成掉帧现象。
- CADisplayLink不能被继承。

- - -
#### NSTimer
基本用法：

```
self.timerInN = [NSTimer timerWithTimeInterval:0.032 target:self selector:@selector(changeImg) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.timerInN forMode:NSRunLoopCommonModes];
```

NSTimer的使用方法也相对简单。

首先，有5个方法可以为我们提供NSTimer实例。
分三类，以timer开头的两个类方法，以schedule开头的两个类方法以及以init开头的一个实例方法。

以timer开头的两个类方法是`灵活度最高`的两个方法。这两个方法的不同点在于绑定事件的方式。一个使用NSInvocation进行转发消息，一个使用target/selector模式绑定事件。总之就是绑定timer的触发事件，这里不做展开讲解。

后面两个参数分别是用户参数以及重复模式。

但是`单单生成了实例还是不会触发我们的事件`，像CADisplayLink一样我们也需要将他`加入到runloop中`,之后就可以触发我们的事件了。

只要是`使用NSTimer就一定要加入到runloop`中才可以触发我们的事件，你可能会说schedule开头那两个类方法就不用添加runloop，这其实是个错觉，是`系统为你将timer添加到了currentRunLoop中，defaultModel`。

最后一个init开头的实例方法就是给timer添加了一个定时启动，这里就不赘述了。

NSTimer还有两个实例方法，`fire和invalid`。分别是立即执行事件和销毁timer。这两个方法比较重要，稍后我会着重讲解一下。

接着说一下他的五个属性。

- `fireDate`，设置当前timer的事件的触发时间。通常我们使用这个属性来做`计时器的暂停与恢复`。

```
///暂停计时器
self.timer.fireDate = [NSDate distantFuture];
///恢复计时器
self.timer.fireDate = [NSDate distantPast];
```

- timeInterval,只读属性，获取`当前timer的事件的触发间隔`。

- tolerance，`允许误差时间`。我们知道`NSTimer事件的触发事件是不准确的`，完全`取决于当前runloop`处理的时间。如果当前runloop在`处理复杂运算`，则timer执行时间`将会被推迟`，直到复杂`运算结束后立即执行触发事件`，之后`再按照初始设置的节奏去执行`。当设置tolerance之后在允许范围内的延迟可以触发事件，超过的则不触发。关于tolerance的设置，苹果有这么一段介绍：

>  As the user of the timer, you will have the best idea of what an appropriate tolerance for a timer may be. A general rule of thumb, though, is to set the tolerance to at least 10% of the interval, for a repeating timer. Even a small amount of tolerance will have a significant positive impact on the power usage of your application. The system may put a maximum value of the tolerance.

翻译成人话就是苹果给了你一个设置`tolerance的参考值`，就是`timeInterval的十分之一`。

- valid，只读属性，获取当前timer是否有效。
- userInfo，用户参数，在初始化的时候传入的用户参数。


说到这里其实NSTimer也就基本介绍完成了，不过老司机还是想着重讲一下NSTimer。

- 关于fire方法
> You can use this method to fire a repeating timer without interrupting its regular firing schedule. If the timer is non-repeating, it is automatically invalidated after firing, even if its scheduled fire date has not arrived.

网上很多人对fire方法的解释其实并不正确。`fire并不是立即激活定时器，而是立即执行一次定时器方法`。当加入到`runloop中timer不需要激活`即可按照设定的时间触发事件。fire只是相当于`手动让timer触发一次事件`。如果timer设置的`repeat为NO，则fire之后timer立即销毁`。如果timer的`repeat为YES`，则到了之前设置的时间他依旧会`按部就班的触发事件`。`fire只是单独触发了一次事件，并不影响原timer的节奏`。


![fire](http://upload-images.jianshu.io/upload_images/1835430-fbd2dbc274695136.gif?imageMogr2/auto-orient/strip)

如上图，默认情况且，根据我写的代码，timerB是不会执行的，应为当前mode并不正确（后面会说）。但是当我点击button也就是执行fire方法时，我们看到timerB响应了事件。

- 关于invalid方法

我们知道NSTimer使用的时候如果不注意的话，是`会造成内存泄漏的`。原因是我们生成实例的时候，会对控制器retain一下。如果不对其进行管理则VC的永远不会引用计数为零，进而造成内存泄漏。

所以，当我们不需要的timer的时候，请如下操作：

```
[self.timer invalid];
self.timer = nil;
```

这样Timer会对VC进行一次release。`所以一定不要忘记调用invalid方法`。

顺便提一句，如果生成timer实例的时候`repeat为NO`，那当触发事件结束后，`系统也会自动调用invalid一次`。

- 关于runloop

有时我们将timer添加到runloop中，而依旧不触发事件。这时候我们应该考虑我们添加到的runloop是否是活跃的runloop。`只有成为活跃的runloop，才会执行runloop中的资源`。


![非活跃runloop](http://upload-images.jianshu.io/upload_images/1835430-4411f0d4977e0e58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 关于mode

即使是目标runloop为活跃runloop依然可能不执行，这时候就要考虑目标runloop是否处于我们指定的mode。`如果不是我们指定的mode，依然不会执行我们的方法`。


![非指定runloopMode](http://upload-images.jianshu.io/upload_images/1835430-664464b0b1457459.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到，我将timerB加入到UITrackingRunLoopMode模式中，默认我们的timerB是不会执行的。因为默认情况下runloop是处于NSDefaultRunLoopMode中的。当scrollView及其子类滚动的时候，runloop会自动切换为追踪模式（UITrackingRunLoopMode）。这是我们的计时器就会工作了。


![切换为正确的Mode](http://upload-images.jianshu.io/upload_images/1835430-dcc29ab0c73b71b6.gif?imageMogr2/auto-orient/strip)

那我们来说一下runloop的几种mode：

- Default模式
 
定义：`NSDefaultRunLoopMode` (Cocoa) kCFRunLoopDefaultMode (Core Foundation)

描述：默认模式中几乎包含了所有输入源(NSConnection除外),一般情况下应使用此模式。

- Connection模式

定义：NSConnectionReplyMode(Cocoa)

描述：处理NSConnection对象相关事件，系统内部使用，用户基本不会使用。

- Modal模式

定义：NSModalPanelRunLoopMode(Cocoa)

描述：处理modal panels事件。

- Event tracking模式

定义：`UITrackingRunLoopMode`(iOS) 
NSEventTrackingRunLoopMode(cocoa)

描述：在拖动loop或其他user interface tracking loops时处于此种模式下，在此模式下会限制输入事件的处理。例如，当手指按住UITableView拖动时就会处于此模式。

- Common模式

定义：`NSRunLoopCommonModes` (Cocoa) kCFRunLoopCommonModes (Core Foundation)

描述：这是一个伪模式，其为一组run loop mode的集合，将输入源加入此模式意味着在Common Modes中包含的所有模式下都可以处理。在Cocoa应用程序中，默认情况下Common Modes包含default modes,modal modes,event Tracking modes.可使用CFRunLoopAddCommonMode方法想Common Modes中添加自定义modes。

**注：iOS中仅NSDefaultRunLoopMode，UITrackingRunLoopMode，NSRunLoopCommonModes三种可用mode。**

你们知道苹果手机为什么崛起的这么快么？第一是因为他是诺基亚年代唯一能与塞班并肩的`智能系统`（毕竟当时用黑莓的很少），当时还没有安卓。第二就是他的`流畅的UI`。

为什么他可以做到UI如德芙一样纵享丝滑呢？因为它赋予了UI极高的地位。全局仅有一条主线程，用来刷新UI。需要不断重绘的scrollView及其子类，享有一个专用的runloopMode，UITrackingRunLoopMode。`当scrollView发生滚动时`，`当前runloop会切换为UITrackingRunLoopMode`。所以正如上面提到过的，如果你的定时器加到NSDefaultRunLoopMode中那么滚动的时候，计时器动作就停止了。这时，你需要`将timer加载NSRunLoopCommonModes`中，才能保证滚动与停止时你的timer都会触发事件。这个对于你的轮播图可是很有用的哦。

这里由于篇幅限制，我并不能展开讲解runloop及mode，建议大家去[这里看看](http://www.cnblogs.com/smileEvday/archive/2012/12/21/NSTimer.html)。

NSTimer的优势：使用相对灵活，应用广泛

劣势：受runloop影响严重，同时易造成内存泄漏（调用invalid方法解决）

- - -
#### dispatch_source_t

其实说dispatch_source_t是timer这样是狭隘的。dispatch_source_t是GCD为我们预留的`源`类型对象。

GCD方法众多，而且各种牛逼的应用，老司机也并不能玩转GCD，所以这里还是主要讲解一下GCD中Timer的用法吧。

```
self.timerInG = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
            dispatch_source_set_timer(self.timerInG,  dispatch_walltime(NULL,0 * NSEC_PER_SEC), 0.032 * NSEC_PER_SEC, 0);
            dispatch_source_set_event_handler(self.timerInG, ^{
                [self changeImg];
            });
```
- dispatch_source_create（,,,） 

这个方法用于返回一个dispatch_source_t对象。第一个参数为源类型，最后一个参数为资源要加入的队列。

- dispatch_source_set_timer(,,,)

这个方法用来设置我们timer的相关信息。第一个参数是我们的timer对象，第二个是timer事件首次触发的延迟时间，第三个参数是timer时间触发的时间间隔，最后一个参数是timer触发的允许延迟值。类比NSTimer的tolerance。建议值也是十分之一。

- dispatch_source_set_event_handler（,）

这个方法用来设置timer的触发事件。第一个参数为Timer对象，第二个为回调block。

- dispatch_resume()

用来激活源对象

- dispatch_suspend()

用来暂停源对象

- dispatch_source_cancel()

用来销毁定时器。

另外需要注意的是，dispatch_source_t  一定要被设置为成员变量，否则将会立即被释放。

关于GCD的timer使用起来相对简单，不过，其实操作不当的话也会造成`内存泄漏`！

`处于挂起`（也就是掉用过 dispatch_suspend()）`的源是不能释放的`。这样就会造成内存泄漏。
所以建议控制器添加一个标识符，记录源是否处于挂起状态，`在dealloc事件中判断当前源是否被挂起，如果被挂起，则resume，即可解决内存泄漏问题`。`同时如果某个源挂起后不需要恢复则直接调用dispatch_source_cancel销毁就好`。

GCDTimer的优势：不`受当前runloopMode的`影响。
劣势：虽然说不受runloopMode的影响，但是其计时效应仍`不是百分之百准确的。另外，他的触发事件也有可能被阻塞，当GCD内部管理的所有线程都被占用时，其触发事件将被延迟`。

- - -

最后，老司机给个demo吧，[点这里](https://pan.baidu.com/s/1jIlp6sA)。
- - -
好了，到这里是不是CADisplayLink说完了=。=其实我是来还债的。

老规矩，求赞，求关注。
