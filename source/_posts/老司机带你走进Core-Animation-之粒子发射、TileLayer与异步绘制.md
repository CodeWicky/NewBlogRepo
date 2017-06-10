---

title: 老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制
layout: post
date: 2017-02-12 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CAAnimation 
- 核心动画
- 粒子效果
- 大图绘制
- 异步绘制
categories: 核心动画

---

![老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://upload-images.jianshu.io/upload_images/1835430-be47acb97de6b6b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


系列文章：

- [老司机带你走进Core Animation 之CAAnimation](http://www.jianshu.com/p/92a0661a21c6)
- [老司机带你走进Core Animation 之CADisplayLink](http://www.jianshu.com/p/434ec6911148)
- [老司机带你走进Core Animation 之几种动画的简单应用](http://www.jianshu.com/p/8e14616679ea)
- [老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://www.jianshu.com/p/3115050b7298)
- [老司机带你走进Core Animation 之图层的透视、渐变及复制](http://www.jianshu.com/p/dedc44fe8e35)
- [老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://www.jianshu.com/p/29cbc1744153)
- - -
呐，今天给大家打来的将是老司机带你走进CoreAnimation系列的最后一篇了，补充一些其他的特殊的layer。

为什么他们放到最后讲呢，因为他们的`使用率不高`，至少在app方面上。所以呢老司机也没怎么用过，`我也是学完又转述给你们`，仅当做`自己学习的笔记了`，所以如果部分内容与您了解的`有偏差`，请`给我留言`，我一定会`与你探讨`，争取`将最正确的博客留给大家`。当然，老司机写这篇博客之前也是自己查阅了很多资料的，你大可以`不用担心我瞎逼逼`╮(╯_╰)╭

![一脸懵逼](http://upload-images.jianshu.io/upload_images/1835430-43e781a3c34ea941.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在今天的博客里，你可以看到以下内容：

- CAEmitterLayer
- CATiledLayer
- 异步绘制

<!-- more -->

- - -

### CAEmitterLayer

CAEmitter的解决粒子发射而存在的类，你问什么是粒子发射，look。

![粒子](http://upload-images.jianshu.io/upload_images/1835430-dec94675e2aca20c.gif?imageMogr2/auto-orient/strip)


你所看到的这一大坨就是粒子系统了。这种效果我们平常app用的还算少点，不过在游戏、直播里面倒是有这不错的应用，所以作为去年两大火热方向之一的直播，你了解一下粒子系统也行。

要想实现这种粒子效果，光有`CAEimmter`还是不够的，他需要`配合着CAEmitterCell进行使用`。要说他们两个的联系呢，就是Layer其实是一个`容器`，而粒子呢就是所谓的`cell`。你可以指定一种粒子样式，也就是一种cell，放在容器layer里面，让容器去控制粒子的发射，就是这样。

东西简单的很，我直接放一下代码，你们看一下就可以懂。

	CAEmitterLayer *emitter = [CAEmitterLayer layer];
    emitter.frame = self.bgView.bounds;
    self.bgView.backgroundColor = [UIColor blackColor];
    [self.bgView.layer addSublayer:emitter];
    ///粒子的产生速率，即每秒产生的个数
    emitter.birthRate = 150;
    ///每个粒子可以停留，也就是显示的时间
    emitter.lifetime = 5;
    ///每个粒子的初速度
    emitter.velocity = 50;
    ///每个粒子的缩放比例
    emitter.scale = 0.5;
    ///每个粒子的旋转角度
    emitter.spin = M_PI;
    ///渲染模式
    emitter.renderMode = kCAEmitterLayerBackToFront;
    ///粒子发生器的位置
    emitter.emitterPosition = CGPointMake(emitter.frame.size.width / 2.0, emitter.frame.size.height / 2.0);
    ///粒子发生器的大小
    emitter.emitterSize = CGSizeMake(50, 50);
    ///粒子发生器的形状
    emitter.emitterShape = kCAEmitterLayerLine;
    ///粒子发射的模式
    emitter.emitterMode = kCAEmitterLayerOutline;
    self.layer = emitter;
    
    //create a particle template
    CAEmitterCell *cell = [[CAEmitterCell alloc] init];
    ///粒子内容，也就是你要展示的粒子元素
    cell.contents = (__bridge id)[UIImage imageNamed:@"爱心"].CGImage;
    ///粒子的初始颜色
    cell.color = [UIColor colorWithRed:0.5 green:0.5 blue:0.5 alpha:1].CGColor;
    
    ///同emitterLayer提供的属性具有相同含义，不同的是emitterLayer中的属性更像是乘法器中的一个比例系数，也就是说layer中的对应属性与cell中的属性相乘才是这个属性最终的值。因为layer可以盛放多种类型cell，所以layer中的数值会作用于所有cell，而cell中的数值仅影响自身实例。
    cell.lifetime = 1;
    cell.birthRate = 1;
    cell.velocity = 1;
    cell.scale = 1;
    cell.spin = 1;
    
    ///一些属性的改变速率，设置了对应值后，粒子的属性会按照改变速率进行改变
    cell.alphaSpeed = -0.1;
    cell.redSpeed = 0.1;
    cell.blueSpeed = 0.1;
    cell.greenSpeed = 0.1;
    
    ///一些属性的扰动范围，所谓扰动范围就是在你设置的固定值两端+-扰动值形成一个范围，那所有粒子的对应属性都在扰动这个范围之内随机产生，这样可以方便产生不同的粒子，而不是千篇一律相同的粒子
    cell.alphaRange = 0.8;
    cell.scaleRange = 1;
    cell.emissionRange = M_PI_4;
    cell.redRange = 1;
    cell.greenRange = 1;
    cell.blueRange = 1;
    
    ///粒子发射倾斜的角度
    cell.emissionLongitude = M_PI_4;
    
    ///粒子y轴运动方向的加速度，相似的还有xAcceleration，通常用来模拟重力加速度，产生粒子坠落的效果
    cell.yAcceleration = 100;
    self.cell = cell;
    
    ///初始化cell后将其放在容器内，容器即可自动产生粒子并进行发射。
    //此处注意是一个数组，也就是说你可以把多个粒子实例传入其中。
    emitter.emitterCells = @[cell];
    
所以说用法还是很简单的，所有属性不同的组合能有一些不错的效果，老司机也就不一一展示了，我的demo里面会抽出几个属性让你能很方便的更改以更快的熟悉CAEmitterLayer。

恩，这里我对发射模式的四个模式讲一下，因为冷不丁一看看不出来头绪

- kCAEmitterLayerPoints 	以端点发射
- kCAEmitterLayerOutline	以发射器边界发射
- kCAEmitterLayerSurface	以发射器内部区域发射
- kCAEmitterLayerVolume		不知道╮(╯_╰)╭

恩，这个就结束了，真不是我敷衍。。就这些东西。。

你可以去[这篇博客](http://www.cnblogs.com/densefog/p/5424155.html)看看每种属性调出来的效果，他都做了动图。


- - - 

### CATiledLayer

这个layer其实你一定见过，至少下面这个效果你一定见过

![Tiled](http://upload-images.jianshu.io/upload_images/1835430-37da6cb408c5a08d.gif?imageMogr2/auto-orient/strip)


这个效果相信用过导航的你就一定见到过吧，如果不用让你自己去实现我相信还会费很大的功夫，不过有了CATiledLayer以后你就不用自己考虑了。

他为什么而存在的呢，就是上面演示那种状况，当你要绘制一幅很大的图片的时候，这将十分耗费性能，因为对于图片的处理我们知道CoreAnimation是强制使用CPU的。所以才有了CATiledLayer。
他将需要绘制的内容`分割成许多小块`，然后再许多线程里`按需异步`绘制相应的小块，这样，就`不会阻塞线程`了。

下面给出部分代码：

```
+(Class)layerClass
{
    return [CATiledLayer class];
}

-(instancetype)initWithFrame:(CGRect)frame path:(NSString *)path completion:(void (^)(CGSize))completion
{
    self = [super initWithFrame:frame];
    if (self) {
        NSURL *docURL = [NSURL fileURLWithPath:path];
        CGPDFDocumentRef pdf = CGPDFDocumentCreateWithURL((CFURLRef)docURL);
        CGPDFPageRef page = CGPDFDocumentGetPage(pdf, 1);
        self.page = page;
        CGRect pageRect = CGPDFPageGetBoxRect(page, kCGPDFCropBox);
        CATiledLayer * layer = (CATiledLayer *)self.layer;
        layer.levelsOfDetailBias = 5;
        
        layer.bounds = pageRect;
        int w = pageRect.size.width;
        int h = pageRect.size.height;
        
        int levels = 1;
        while (w > 1 && h > 1) {
            levels++;
            w = w >> 1;
            h = h >> 1;
        }
        layer.levelsOfDetail = levels;
        [self setFrame:pageRect];
        if (completion) {
            completion(pageRect.size);
        }
    }
    return self;
}

-(void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx
{
    CGContextDrawPDFPage(ctx, self.page);
}
```

这个就是上面demo中我重写的view类。

有两个属性说一下：

- levelsOfDetail：layer维护的细节层次的个数，就是说你对图像进行2倍缩放而不失真的层次的个数。

- levelsOfDetailBias：layer维护的细节层次放大倍数。

这两个地方很抽象，老司机也用的比较少，老司机一知半解的也不好误导你，所以你可以看看这几个博客，都有介绍这个属性：

- [
Swift语言iOS开发：CALayer十则示例](http://mobile.51cto.com/iphone-469498.htm)

- [CATiledLayer (Part 2)](http://www.mlsite.net/blog/?p=1884)

- [研究了一下CATiledLayer的levelsOfDetail和levelsOfDetailBias的含义](http://www.cocoachina.com/bbs/read.php?tid-31201-page-1.html)

- - -
### 异步绘制

下面我会着重讲一下异步绘制。（毕竟这才是最近用到的深入了解过的东西）。

最初这个想法是从`ASDK`来的。
总的来说`ASDK`是FaceBook`为了解决iOS中由于计算量过多而导致屏幕卡顿`的一个开源库。

我们平时感受到的卡顿，其实专业点叫掉帧（玩游戏的你一定知道）。造成掉帧是因为有垂直同步，当计算量过大导致要刷新帧的时候没有计算结束，就会放弃本帧，造成了所谓了卡顿。（更多关于成像原理的内容去[这里](http://draveness.me/asdk-rendering/)看）。

所以知道了卡顿的原因，我们就要想办法解决这当中的冲突，所以我们应该了解CoreAnimation的`绘制流程`。

我们知道实际上CALayer和UIView都`不是线程安全`的，所以UI操作我们一定要写在主线程（虽然后来苹果也修改了一部分属性使其成为线程安全的，但是苹果仍不建议在子线程中操作UI，因为你无法预知会发生什么。。）

说一下你对UI的改变系统是怎么处理的。
我们知道，OS及iOS系统中，`负责渲染的类均为CALayer类`，也就是说，你操作的所有UI，layer层也好UI控件也罢，`最后都会映射到CALayer`的改变上。

但是Layer捕捉到需要做的改变后`并不会立即去刷新`，而是`寻找一个合适的实际去进行刷新`。

事实上CoreAnimation在Runloop中注册了一个`观察者`，当runLoop即将进入`休眠或者退出`的时候会回调，这时候CALayer捕捉的到所有变化会开始计算，并刷新UI。

前文说过，早成屏幕卡顿的原因是因为`计算量大`，没算完，掉帧了。那我们为什么不在他接收到需要变化的时候`尽可能的将可以在子线程中做的计算开启一个子线程异步去计算`，其他的回到中线程中计算，这样`减少计算压力`就可以提高性能。

上面说了一下原理以及由原理衍生出来的解决方案，下面就具体放一下代码。这里需要说的是由于ASDK太过庞大，我要讲的是`YYKit`中对于异步绘制这部分的代码，恩，他是参考的ASDK，所以原理一样。以下代码是我参照`YYTextAsyncLayer`改的代码，因为YY大神代码的完备性，所以我也没有什么改善的余地，只是将个人认为不需要的冗余去除，进行了微乎其微的改动，所以这里申明代码`不是我个人代码`啊。。。

```
static dispatch_queue_t DWCoreTextLabelLayerGetDisplayQueue() {
#define MAX_QUEUE_COUNT 16
    static int queueCount;
    static dispatch_queue_t queues[MAX_QUEUE_COUNT];
    static dispatch_once_t onceToken;
    static int32_t counter = 0;
    dispatch_once(&onceToken, ^{
        queueCount = (int)[NSProcessInfo processInfo].activeProcessorCount;
        queueCount = queueCount < 1 ? 1 : queueCount > MAX_QUEUE_COUNT ? MAX_QUEUE_COUNT : queueCount;
        if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0) {
            for (NSUInteger i = 0; i < queueCount; i++) {
                dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, 0);
                queues[i] = dispatch_queue_create("com.codeWicky.DWCoreTextLabel.render", attr);
            }
        } else {
            for (NSUInteger i = 0; i < queueCount; i++) {
                queues[i] = dispatch_queue_create("com.codeWicky.DWCoreTextLabel.render", DISPATCH_QUEUE_SERIAL);
                dispatch_set_target_queue(queues[i], dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));
            }
        }
    });
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    uint32_t cur = (uint32_t)OSAtomicIncrement32(&counter);
#pragma clang diagnostic pop
    return queues[(cur) % queueCount];
#undef MAX_QUEUE_COUNT
}

static dispatch_queue_t DWCoreTextLabelLayerGetReleaseQueue() {
    return dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
}

@interface DWAsyncLayer ()

@property (atomic, readonly) int32_t signal;

@end

@implementation DWAsyncLayer

-(instancetype)init
{
    self = [super init];
    if (self) {
        _signal = 0;
        _displaysAsynchronously = YES;
    }
    return self;
}

-(void)signalIncrease
{
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    OSAtomicIncrement32(&_signal);
#pragma clang diagnostic pop
}

-(void)setNeedsDisplay
{
    [self cancelPreviousDisplayCalculate];
    [super setNeedsDisplay];
}

-(void)cancelPreviousDisplayCalculate
{
    [self signalIncrease];
}

-(void)dealloc
{
    [self cancelPreviousDisplayCalculate];
}

-(void)display
{
    super.contents = super.contents;
    [self displayAsync:self.displaysAsynchronously];
}

-(void)displayAsync:(BOOL)async
{
    if (!self.displayBlock) {
        self.contents = nil;
        return;
    }
    if (async) {
        int32_t signal = self.signal;
        BOOL (^isCancelled)() = ^BOOL() {
            return signal != self.signal;
        };
        CGSize size = self.bounds.size;
        BOOL opaque = self.opaque;
        CGFloat scale = self.contentsScale;
        CGColorRef backgroundColor = (opaque && self.backgroundColor) ? CGColorRetain(self.backgroundColor) : NULL;
        if (size.width < 1 || size.height < 1) {
            CGImageRef image = (__bridge_retained CGImageRef)(self.contents);
            self.contents = nil;
            if (image) {
                dispatch_async(DWCoreTextLabelLayerGetReleaseQueue(), ^{
                    CFRelease(image);
                });
            }
            CGColorRelease(backgroundColor);
            return;
        }
        
        dispatch_async(DWCoreTextLabelLayerGetDisplayQueue(), ^{
            if (isCancelled()) {
                CGColorRelease(backgroundColor);
                return;
            }
            UIGraphicsBeginImageContextWithOptions(size, opaque, scale);
            CGContextRef context = UIGraphicsGetCurrentContext();
            if (opaque) {
                fillContextWithColor(context, backgroundColor, size);
                CGColorRelease(backgroundColor);
            }
            self.displayBlock(context,isCancelled);
            if (isCancelled()) {
                UIGraphicsEndImageContext();
                return;
            }
            UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
            if (isCancelled()) {
                return;
            }
            dispatch_async(dispatch_get_main_queue(), ^{
                if (!isCancelled()) {
                    self.contents = (__bridge id)(image.CGImage);
                }
            });
        });
    } else {
        [self signalIncrease];
        UIGraphicsBeginImageContextWithOptions(self.bounds.size, self.opaque, self.contentsScale);
        CGContextRef context = UIGraphicsGetCurrentContext();
        if (self.opaque) {
            CGSize size = self.bounds.size;
            size.width *= self.contentsScale;
            size.height *= self.contentsScale;
            fillContextWithColor(context, self.backgroundColor,size);
        }
        self.displayBlock(context,^{return NO;});
        UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        self.contents = (__bridge id)(image.CGImage);
    }
}

static inline void fillContextWithColor(CGContextRef context,CGColorRef color,CGSize size){
    CGContextSaveGState(context); {
        if (!color || CGColorGetAlpha(color) < 1) {
            CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
            CGContextAddRect(context, CGRectMake(0, 0, size.width, size.height));
            CGContextFillPath(context);
        }
        if (color) {
            CGContextSetFillColorWithColor(context, color);
            CGContextAddRect(context, CGRectMake(0, 0, size.width, size.height));
            CGContextFillPath(context);
        }
    } CGContextRestoreGState(context);
};

@end
```

总共不到200行代码，因为我减少了YY里面的将要绘制和完成绘制的回调，仅保留绘制的回调。

大致说一下他的思路：

当一次绘制请求正在进行的时候，如果下一次绘制请求已经开始，那么显然本次请求是无效的，应当放弃，所以这里`一次绘制应该尽可能的及早被取消`。而后就是`将无关的计算移到子线程`中计算`并在context中绘制`，最后从`context中取到绘制的图片`，将其设置为`layer的Contents进而展示画面`。

所以思路在这，实现方式就出来了：

- 1.截获绘制请求，进行自定制实现

- 2.实现过程中，可以取消绘制请求

- 3.将绘制任务在子线程中回调给出去进行绘制，再取context中Image在主线程中设置给contents。

恩，第一条不用说了，绘制任务会调用layer的`display`方法，`重写`就好了。

第二条分两点，`发现`取消请求和`取消任务`。取消任务比较简单，在将要执行的代码前`return`就会导致后面的代码不被执行进而取消。

巧妙地是发现取消请求这里。我们知道`block引入变量的时候会将外界变量copy于栈`中，这样即使外界变量发生改变，block中的变量也不会发生改变。（当然只有基本类型数据传入的是值的，对象都是传指针的。这部分的理解你要参照`传值与传止的区别，以及形参与实参`，真不懂的童靴回头复习一下C吧）。应用这一特性，设置Layer持有一个`基本数据类型的计数量`，`用一个临时变量存储及数量后`，block中比较临时变量与layer持有的计数量的值，因为临时变量是被copy走的，不会随外界改变，所以当外界改变时，`两个值不相等我们就能拿到这个状态了`。

然后就是当我们要`取消任务的时候更改layer持有的计数量`，从而取消任务。

第三条，就是在具体需要绘制的地方，（一般来说为了`降低耦合性都是扔给View去做`，这样所有的View的layer都使用这个layer，而不同的绘制任务交给View中layer的回调即可解耦）把绘制任务都绘制在context，绘制完成后在将context的image在主线程中赋给contents即可。（说到这看不懂的建议看一下这系列的第一篇，一些预备知识都在里面）。

到这里，异步绘制这块也算是结束了。
- - -

此外，单独说一句，老司机之前借鉴这个思路后想着一些其他实现方式拿到context也可以做这些是，不过都以失败告终，为什么呢，因为`除了drawRect中，UIView里面是拿不到context`的，这是因为drawRect方法是在drawLayer:inContext中进行调用，并且调用之前会执行UIGraphicsPushContext(context)，将当前context压入栈顶，这时你在drawRect中才能通过UIGraphicsGetCurrentContext取到。并且drawRect执行完成后，context紧接着呗pop掉了，所以你只能在drawRect中获取到当前context。

- - -
本期的demo[在这里](https://pan.baidu.com/s/1hrW8hQs)。
- - - 

恩，到这里终于能结束这个系列了。随着越来越多的人看我的博客，老司机也开始觉得话不能乱说，字不能乱码。所以老司机也正在努力不断的提升自己博客中的内容质量，争取对得起美味粉丝~😯，不，是每位粉丝，毕竟老司机不C粉。

![你猜我笑啥](http://upload-images.jianshu.io/upload_images/1835430-1a72586af69776ec.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- - -
参考资料

- [iOS粒子系统CAEmitterLayer](http://www.cnblogs.com/densefog/p/5424155.html)
- [研究了一下CATiledLayer的levelsOfDetail和levelsOfDetailBias的含义](http://www.cocoachina.com/bbs/read.php?tid-31201-page-1.html)
- [使用 ASDK 性能调优 - 提升 iOS 界面的渲染性能](http://draveness.me/asdk-rendering/)
- [iOS 保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
- [关于drawRect](http://www.jianshu.com/p/c49833c04362)

- - -


最后的最后，一般都是软广环节：

DWCoreTextLabel更新到现在已经1.1.6版本了，现在除了图文混排功能，还支持文本类型的自动检测，异步绘制减少系统的卡顿，异步加载并缓存图片的功能。

>version 1.1.0
>全面支持自动链接支持、定制检测规则、图文混排、响应事件
>优化大部分算法，提高响应效率及绘制效率
 
>version 1.1.1
>高亮取消逻辑优化
>自动检测逻辑优化
>部分常用方法改为内联函数，提高运行效率
 
>version 1.1.2
>绘制逻辑优化，改为异步绘制（源码修改自YYTextAsyncLayer）
 
>version 1.1.3
>异步绘制改造完成、去除事务管理类，事务管理类仍可改进，进行中
 
>version 1.1.4
>事务管理类去除，异步绘制文件抽出
 
>version 1.1.5
>添加网络图片异步加载库，支持绘制网络图片
>

![DWCoreTextLabel](http://upload-images.jianshu.io/upload_images/1835430-bc004dc959f7b675.gif?imageMogr2/auto-orient/strip)


插入图片、绘制图片、添加事件统统一句话实现~


![一句话实现](http://upload-images.jianshu.io/upload_images/1835430-456beaed3f710b02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


尽可能保持系统Label属性让你可以无缝过渡使用~


![无缝过渡](http://upload-images.jianshu.io/upload_images/1835430-e7706ffe827b1ea1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


恩，说了这么多，老司机放一下地址：[DWCoreTextLabel](https://github.com/CodeWicky/DWCoreTextLabel)，宝宝们给个star吧~爱你哟~


![爱你哟](http://upload-images.jianshu.io/upload_images/1835430-1a477250d897b4b1.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
