---

title: 老司机带你走进Core Animation 之图层的透视、渐变及复制
layout: post
date: 2016-12-21 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CAAnimation 
- 核心动画
- 图层效果
categories: 核心动画

---

![老司机带你走进Core Animation 之图层的透视、渐变及复制](http://upload-images.jianshu.io/upload_images/1835430-deee60e266a22d65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：

- [老司机带你走进Core Animation 之CAAnimation](http://www.jianshu.com/p/92a0661a21c6)
- [老司机带你走进Core Animation 之CADisplayLink](http://www.jianshu.com/p/434ec6911148)
- [老司机带你走进Core Animation 之几种动画的简单应用](http://www.jianshu.com/p/8e14616679ea)
- [老司机带你走进Core Animation 之CAShapeLayer和CATextLayer](http://www.jianshu.com/p/3115050b7298)
- [老司机带你走进Core Animation 之图层的透视、渐变及复制](http://www.jianshu.com/p/dedc44fe8e35)
- [老司机带你走进Core Animation 之粒子发射、TileLayer与异步绘制](http://www.jianshu.com/p/29cbc1744153)
- - -
这回呢，当然还是顺着头文件里面的几个类，老司机一个一个捋吧。

老司机的想法就是要把CoreAnimation头文件中的类大概都说一遍，毕竟一开始把系列名定成了《老司机带你走进CoreAnimation》（深切的觉得自己给自己坑了。。。）。
![我给自己挖的坑](http://upload-images.jianshu.io/upload_images/1835430-ff640963309ad310.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以呢，在今天的博客里你将会看到以下截个内容

- CATransform3D
- CATransformLayer
- CAGradientLayer
- CAReplicatorLayer
- DWMirrorView

废话不多说，直接进入主题。

<!-- more -->

- - -

### CATransform3D

先介绍一下CATransform3D吧。

![CATransform3D](http://upload-images.jianshu.io/upload_images/1835430-84870af23882317f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

正如上图所示，我们可以清晰的看到，CATransform3D是一个结构体。而且苹果很友好的调整了一下书写格式，正如你看到的，它像是一个4 X 4的矩阵。

事实上他的原理就是一个`4 X 4矩阵`。

其实他还有一个弟弟，CGAffineTransform。这是一个3 X 3的矩阵。
他们的作用都一样，进行坐标变换。
不同点在于，CATransform3D作用与3维坐标系的坐标变换，CGAffineTransform作用于2维坐标系的坐标变换。

所以CGAffineTransform用于对UIView进行变换，而CATransform3D用于对CALayer进行变换。

虽然老司机从小到大都是数学课代表，不过我要很郑重的告诉你，数学是一门靠悟性的学问，不是我讲给你听，你就能消化的，所以关于矩阵计算什么的，请各位同学自己消化理解（咳咳，我会告诉你我高数、线代、概率没有一科过70的么=。=）
![一脸无辜](
http://upload-images.jianshu.io/upload_images/1835430-60aec696da69627f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
)

所以呢，老司机直接来介绍CATransform3D的相关api吧。（CGAffineTransform的api与CATransform3D相似，可类比使用）。

> - CATransform3DIdentity
> 
>生成一个无任何变换的默认矩阵，可用于使变换后的Layer恢复初始状态
>
>- - -
> - CATransform3DMakeTranslation
> - CATransform3DMakeScale 
> - CATransform3DMakeRotation
> 
> 分别是平移、缩放、旋转，这三个api最大的相同点就在于函数名中都有Make。意思就是生成指定变换的矩阵。与之相对的就是下面的api👇
> 
> - CATransform3DTranslate
> - CATransform3DScale
> - CATransform3DRotate
> 
> 与之前三个api的不同点在于，这三个api都多了一个参数，同样是一个CATransform3D结构体。我想你一定猜到了，就是对给定的矩阵在其现有基础上进行指定的变换。
> 
> 值得注意的是，以上两个旋转api中x/y/z三个参数均为指定旋转轴，可选值0和1，`0代表此轴不做旋转`，`1代表作旋转`。例如想对x、y轴做45度旋转，则angle = M____PI____4,x = 1,y = 1,z = 0。另外，此处旋转角度为`弧度制`哦，不是角度制。
> 
> - - -
> - CATransform3DConcat
> 
> 返回两个矩阵的乘积。
> 
> - - -
> 
> - CATransform3DInvert
> 
> 反转矩阵
> 
> - - -
> 
> - CATransform3DMakeAffineTransform
> - CATransform3DGetAffineTransform
> 
> CGAffineTransform与CATransform3D相互转化的api
> 
> - - -
> 
> - CATransform3DIsIdentity
> - CATransform3DEqualToTransform
> - CATransform3DIsAffine
> 
> 这三个api见名知意了，不多说。
> 

哦，重要的一点你一定要知道，所有的矩阵变换都是相对于图层的`锚点`进行的。还记得锚点的概念么？不记得可以去这个系列的[第一篇文章](http://www.jianshu.com/p/92a0661a21c6)补课哦。

其实呢，关于CATransform3D你只要会使用以上api对图层做3维坐标转换就够了。不过上述这些变换默认情况下都是`不具备透视效果`的，因为你所看到的都是图层`在x轴y轴上的投影`，那想要透视效果怎么办呢？两个办法，CATranformLayer，以及M34。

#### M34

老司机说过，CATransform3D不过是一个4 X 4的矩阵。那么其实这16个数字中，每一个数字都有自己可以控制的转换，这是纯数学知识啊，自己领悟=。=不过老司机可以单独说说`M34`这个数。这个数是用来控制图层变换后的`景深效果`的，也就是`透视效果`。

![M34](http://upload-images.jianshu.io/upload_images/1835430-5f03c502308df4c2.gif?imageMogr2/auto-orient/strip)

上面的图片分别展示了具有透视效果的旋转及动画。

代码上的体现就是

```
	CALayer * staticLayerA = [CALayer layer];
    staticLayerA.bounds = CGRectMake(0, 0, 100, 100);
    staticLayerA.position = CGPointMake(self.view.center.x - 75, self.view.center.y - 100);
    staticLayerA.backgroundColor = [UIColor redColor].CGColor;
    [self.view.layer addSublayer:staticLayerA];
    
    CATransform3D transA = CATransform3DIdentity;
    transA.m34 = - 1.0 / 500;
    transA = CATransform3DRotate(transA, M_PI / 3, 1, 0, 0);
    staticLayerA.transform = transA;

```

使用上很简单，代码里M34 = - 1.0 / 500 的意思就是`图层距离屏幕向里500的单位`。如果向外则是M34 = 1.0 / 500。这个距离至一般掌握至500~1000这个范围会取得不错的效果。

这里需要注意的是`M34的赋值一定要写在矩阵变换前面`，具体为什么说实话老司机也不知道。

- - -

### CATransformLayer

老司机上面提到过，CALayer做矩阵变换你能看到的只是他在XY轴上的投影，这时你若想看到透视效果，就需要使用到M34或CATransformLayer。其实他两个又有一些区别，CATransformLayer是让你看到的不只是其在XY轴上的投影。

说起来不好懂，看下面的图吧。

![TransformLayer](http://upload-images.jianshu.io/upload_images/1835430-6a1cec404c39345f.gif?imageMogr2/auto-orient/strip)

CATransformLayer可以让其的`子视图各自现实自身的真实形状，而不是其在父视图的投影`。

你可能还不懂，其实你看的正方体是六个CALayer经过矩阵变换拼成的实实在在的正方体。

```
	//create cube layer
    CATransformLayer *cube = [CATransformLayer layer];
    
    //add cube face 1
    CATransform3D ct = CATransform3DMakeTranslation(0, 0, 50);
    [cube addSublayer:[self faceWithTransform:ct]];
    
    //add cube face 2
    ct = CATransform3DMakeTranslation(50, 0, 0);
    ct = CATransform3DRotate(ct, M_PI_2, 0, 1, 0);
    [cube addSublayer:[self faceWithTransform:ct]];
    
    //add cube face 3
    ct = CATransform3DMakeTranslation(0, -50, 0);
    ct = CATransform3DRotate(ct, M_PI_2, 1, 0, 0);
    [cube addSublayer:[self faceWithTransform:ct]];
    
    //add cube face 4
    ct = CATransform3DMakeTranslation(0, 50, 0);
    ct = CATransform3DRotate(ct, -M_PI_2, 1, 0, 0);
    [cube addSublayer:[self faceWithTransform:ct]];
    
    //add cube face 5
    ct = CATransform3DMakeTranslation(-50, 0, 0);
    ct = CATransform3DRotate(ct, -M_PI_2, 0, 1, 0);
    [cube addSublayer:[self faceWithTransform:ct]];
    
    //add cube face 6
    ct = CATransform3DMakeTranslation(0, 0, -50);
    ct = CATransform3DRotate(ct, M_PI, 0, 1, 0);
    [cube addSublayer:[self faceWithTransform:ct]];
    
    //center the cube layer within the container
    CGSize containerSize = self.containerView.bounds.size;
    cube.position = CGPointMake(containerSize.width / 2.0, containerSize.height / 2.0);
```
```
- (CALayer *)faceWithTransform:(CATransform3D)transform
{
    //create cube face layer
    CALayer *face = [CALayer layer];
    face.bounds = CGRectMake(0, 0, 100, 100);
    //apply a random color
    CGFloat red = (rand() / (double)INT_MAX);
    CGFloat green = (rand() / (double)INT_MAX);
    CGFloat blue = (rand() / (double)INT_MAX);
    face.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;
    face.transform = transform;
    return face;
}

```
使用起来就是这么简单，把各个变换后的layer加入到CATransformLayer中就可以了。

本身CATransformLayer不具有任何其他属性，其实他更像是`一个容器`。它本身至渲染其子图层，自身没有任何layer的属性。

最重要的一点是，当图层加入到CATransformLayer中以后，`hitTest和convertPoint两个方法就失效了`，请注意这点。

- - -

### CAGradientLayer

CAGradientLayer本身的属性也比较少，而且完全是针对于过渡颜色来的。

> - colors
> 
> 图层显示的所有颜色的数组
> 
> - - -
> - locations
> 
> 每个颜色对应的位置。注意，这个位置指的是颜色的位置，而不是过渡线的位置。
> 
> - - - 
> - startPoint
> - endPoint
> 
> 是颜色过渡的方向，会沿着起点到终点的向量进行过渡。
> - - -
> 
> - type
> 
> 过渡模式，当前苹果给我们暴露的只有一种模式，kCAGradientLayerAxial。
> 

需要说明的是，CAGradientLayer只能做`矩形的渐变图层`。

![你要怎么做？](http://upload-images.jianshu.io/upload_images/1835430-ed19e2d814f232f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


所以说这个效果要如何实现呢？其实啊，这只是一个错觉，看这个。

![矩形渐变层](http://upload-images.jianshu.io/upload_images/1835430-cd67967da4814962.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以说看到这你就知道了吧，两个拼一起的CAGradientLayer，然后用一个shapeLayer做了一个mask就成了环形的过渡层了。这一招老司机早就做了过，还记得么，[歌词显示那一章](http://www.jianshu.com/p/8e14616679ea)。

```
- (void)viewDidLoad {
    [super viewDidLoad];

    UIBezierPath * path = [UIBezierPath bezierPathWithArcCenter:CGPointMake(50, 50) radius:45 startAngle:- 7.0 / 6 * M_PI endAngle:M_PI / 6 clockwise:YES];
    
    [self.view.layer addSublayer:[self createShapeLayerWithPath:path]];
    
    CAGradientLayer * leftL = [self createGradientLayerWithColors:@[(id)[UIColor redColor].CGColor,(id)[UIColor yellowColor].CGColor]];
    leftL.position = CGPointMake(25, 40);
    
    CAGradientLayer * rightL = [self createGradientLayerWithColors:@[(id)[UIColor greenColor].CGColor,(id)[UIColor yellowColor].CGColor]];
    rightL.position = CGPointMake(75, 40);
    
    CALayer * layer = [CALayer layer];
    layer.bounds = CGRectMake(0, 0, 100, 80);
    layer.position = self.view.center;
    [layer addSublayer:leftL];
    [layer addSublayer:rightL];
    [self.view.layer addSublayer:layer];
    
    CAShapeLayer * mask = [self createShapeLayerWithPath:path];
    mask.position = CGPointMake(50, 40);
    layer.mask = mask;
    mask.strokeEnd = 0;
    self.mask = mask;
}

-(CAShapeLayer *)createShapeLayerWithPath:(UIBezierPath *)path
{
    CAShapeLayer * layer = [CAShapeLayer layer];
    layer.path = path.CGPath;
    layer.bounds = CGRectMake(0, 0, 100, 75);
    layer.position = self.view.center;
    layer.fillColor = [UIColor clearColor].CGColor;
    layer.strokeColor = [UIColor colorWithRed:33 / 255.0 green:192 / 255.0 blue:250 / 255.0 alpha:1].CGColor;
    layer.lineCap = @"round";
    layer.lineWidth = 10;
    return layer;
}

-(CAGradientLayer *)createGradientLayerWithColors:(NSArray *)colors
{
    CAGradientLayer * gradientLayer = [CAGradientLayer layer];
    gradientLayer.colors = colors;
    gradientLayer.locations = @[@0,@0.8];
    gradientLayer.startPoint = CGPointMake(0, 1);
    gradientLayer.endPoint = CGPointMake(0, 0);
    gradientLayer.bounds = CGRectMake(0, 0, 50, 80);
    return gradientLayer;
}
```

- - -

### CAReplicatorLayer
CAReplicatorLayer官方的解释是一个`高效处理复制图层的中间层`。他能复制图层的`所有属性`，`包括动画`。

使用起来很简单，从他的属性一一看：

> - instanceCount
> 
> 实例数，复制后的实例数。
> 
> - preservesDepth
> 
> 这是一个bool值，默认为No，如果设为Yes，将会具有3维透视效果。
> 
> - instanceDelay
> - instanceTransform
> - instanceColor
> - instanceRedOffset
> - instanceGreenOffset
> - instanceBlueOffset
> - instanceAlphaOffset
> 
> 这一排属性都是每一个实例与上一个实例对应属性的偏移量。
> 

还是上代码吧，直观点。

```
//     Do any additional setup after loading the view.
        CALayer * layer = [CALayer layer];
        layer.bounds = CGRectMake(0, 0, 30, 30);
        layer.position = CGPointMake(self.view.center.x - 50, self.view.center.y - 50);
        layer.backgroundColor = [UIColor redColor].CGColor;
        layer.cornerRadius = 15;
        [self.view.layer addSublayer:layer];
    
        CABasicAnimation * animation1 = [CABasicAnimation animationWithKeyPath:@"opacity"];
        animation1.fromValue = @(0);
        animation1.toValue = @(1);
        animation1.duration = 1.5;
    //    animation1.autoreverses = YES;
    
        CABasicAnimation * animation2 = [CABasicAnimation animationWithKeyPath:@"transform.scale"];
        animation2.toValue = @(1.5);
        animation2.fromValue = @(0.5);
        animation2.duration = 1.5;
    //    animation2.autoreverses = YES;
    
        CAAnimationGroup * ani = [CAAnimationGroup animation];
        ani.animations = @[animation1,animation2];
        ani.duration = 1.5;
        ani.repeatCount = MAXFLOAT;
        ani.autoreverses = YES;
    
        [layer addAnimation:ani forKey:nil];
    
        CAReplicatorLayer * rec = [CAReplicatorLayer layer];
        [rec addSublayer:layer];
        rec.instanceCount = 3;
        rec.instanceDelay = 0.5;
        rec.instanceTransform = CATransform3DMakeTranslation(50, 0, 0);
        [self.view.layer addSublayer:rec];
    
        CAReplicatorLayer * rec2 = [CAReplicatorLayer layer];
        [rec2 addSublayer:rec];
        rec2.instanceCount = 3;
        rec2.instanceDelay = 0.5;
        rec2.instanceTransform = CATransform3DMakeTranslation(0, 50, 0);
        [self.view.layer addSublayer:rec2];
```
正如你所看到的，`CAReplicatorLayer支持嵌套使用`。它的效果是如下这个样子的。

![ReplicatorLayer](http://upload-images.jianshu.io/upload_images/1835430-cdc45f4b556ba1b5.gif?imageMogr2/auto-orient/strip)

- - -

### DWMirrorView

啧啧啧，没想到今天的内容就这么讲完了。

额，内容比较少，的确是今天讲的这几个比较简单。我知道只是这样你们是不会放过我的。那我就放一个这几个属性联合起来的一个小应用吧。

![Mirror.gif](http://upload-images.jianshu.io/upload_images/1835430-72af1c07bb22ff51.gif?imageMogr2/auto-orient/strip)

忽略倒影的层次感吧，截图问题，正常是一个梯度渐变下去的。

其实拿到一个需求，我们先分析一下想要实现他的步骤，这个过程对于开发其实是很重要的。

>首先来说，我们看到的倒影，我们应该可以考虑CAReplicator做一个复制图层，配合instranceTransform属性做出倒影效果

>然后来说，我们看到了倒影渐变效果，我们应该想到的是使用CAGradientLayer去实现过渡效果。

>最后一些细节的参数我们可以根据需求去进行相关设置。

分析过后其实思路还是挺清晰的，一步步实现就好。
实现起来很简单，代码量也不多，我就直接放代码就好了。

```
#pragma mark ---tool method---

-(void)handleMirrorDistant:(CGFloat)distant
{
    CAReplicatorLayer * layer = (CAReplicatorLayer *)self.layer;
    CATransform3D transform = CATransform3DIdentity;
    transform = CATransform3DTranslate(transform, 0, distant + self.bounds.size.height, 0);
    transform = CATransform3DScale(transform, 1, -1, 0);
    layer.instanceTransform = transform;
}

-(NSArray *)getMaskLayerLocations
{
    CGFloat height = self.bounds.size.height * 2 + self.mirrorDistant;
    CGFloat mirrowScale = self.bounds.size.height * (1 + self.mirrorScale) + self.mirrorDistant;
    return @[@0,@((self.bounds.size.height + self.mirrorDistant) / height),@(mirrowScale / height)];
}

-(CGFloat)safeValueBetween0And1:(CGFloat)value
{
    if (value > 1) {
        value = 1;
    } else if (value < 0) {
        value = 0;
    }
    return value;
}

-(void)valueInit
{
    self.mirrorDistant = 0;
    self.mirrorScale = 0.5;
    self.mirrored = YES;
    self.dynamic = YES;
    self.mirrorAlpha = 0.5;
}

#pragma mark ---override---

-(instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        [self valueInit];
    }
    return self;
}

-(void)awakeFromNib
{
    [super awakeFromNib];
    [self valueInit];
}

+(Class)layerClass
{
    return [CAReplicatorLayer class];
}

-(void)drawRect:(CGRect)rect
{
    [super drawRect:rect];
    CAReplicatorLayer * layer = (CAReplicatorLayer *)self.layer;
    if (self.mirrored) {
        if (self.dynamic) {
            [self.mirrorImageView removeFromSuperview];
            self.mirrorImageView = nil;
            layer.instanceCount = 2;
            if (CATransform3DEqualToTransform(layer.instanceTransform, CATransform3DIdentity)) {
                [self handleMirrorDistant:self.mirrorDistant];
            }
        }
        else
        {
            layer.instanceCount = 1;
            CGSize size = CGSizeMake(self.bounds.size.width, self.bounds.size.height * self.mirrorScale);
            if (size.height > 0.0f && size.width > 0.0f)
            {
                UIGraphicsBeginImageContextWithOptions(size, NO, 0.0f);
                CGContextRef context = UIGraphicsGetCurrentContext();
                CGContextScaleCTM(context, 1.0f, -1.0f);
                CGContextTranslateCTM(context, 0.0f, -self.bounds.size.height);
                [self.layer renderInContext:context];
                self.mirrorImageView.image = UIGraphicsGetImageFromCurrentImageContext();
                UIGraphicsEndImageContext();
            }
            self.mirrorImageView.alpha = self.mirrorAlpha;
            self.mirrorImageView.frame = CGRectMake(0, self.bounds.size.height + self.mirrorDistant, size.width, size.height);
        }
        self.layer.mask = self.maskLayer;
    }
    else
    {
        layer.instanceCount = 1;
        [self.mirrorImageView removeFromSuperview];
        self.mirrorImageView = nil;
        self.layer.mask = nil;
    }
    
}

#pragma mark ---setter/getter---

-(void)setMirrored:(BOOL)mirrored
{
    _mirrored = mirrored;
    [self setNeedsDisplay];
}

-(void)setDynamic:(BOOL)dynamic
{
    _dynamic = dynamic;
    [self setNeedsDisplay];
}

-(void)setMirrorAlpha:(CGFloat)mirrorAlpha
{
    _mirrorAlpha = [self safeValueBetween0And1:mirrorAlpha];
    if (self.mirrored) {
        if (self.dynamic) {
            CAReplicatorLayer * layer = (CAReplicatorLayer *)self.layer;
            layer.instanceAlphaOffset = self.mirrorAlpha - 1;
        }
        else
        {
            [self setNeedsDisplay];
        }
    }
    
}

-(void)setMirrorScale:(CGFloat)mirrorScale
{
    _mirrorScale = [self safeValueBetween0And1:mirrorScale];
    if (self.mirrored) {
        self.maskLayer.locations = [self getMaskLayerLocations];
        if (!self.dynamic) {
            [self setNeedsDisplay];
        }
    }
}

-(void)setMirrorDistant:(CGFloat)mirrorDistant
{
    _mirrorDistant = mirrorDistant;
    if (self.mirrored) {
        self.maskLayer = nil;
        [self handleMirrorDistant:mirrorDistant];
        [self setNeedsDisplay];
    }
}

-(CAGradientLayer *)maskLayer
{
    if (!_maskLayer) {
        _maskLayer = [CAGradientLayer layer];
        _maskLayer.frame = CGRectMake(0, 0, self.bounds.size.width, self.bounds.size.height * 2 + self.mirrorDistant);
        _maskLayer.startPoint = CGPointMake(0, 0);
        _maskLayer.endPoint = CGPointMake(0, 1);
        _maskLayer.locations = [self getMaskLayerLocations];
        _maskLayer.colors = @[(id)[UIColor blackColor].CGColor,(id)[UIColor blackColor].CGColor,(id)[UIColor clearColor].CGColor];
    }
    return _maskLayer;
}

-(UIImageView *)mirrorImageView
{
    if (!_mirrorImageView) {
        _mirrorImageView = [[UIImageView alloc] initWithFrame:self.bounds];
        _mirrorImageView.contentMode = UIViewContentModeScaleToFill;
        _mirrorImageView.userInteractionEnabled = NO;
        [self addSubview:_mirrorImageView];
    }
    return _mirrorImageView;
}
```

因为本身只作为一个容器存在，不需要对外界留一些接口，所以总共才200行代码，不过实现的效果还是可以的。

- - -

今天的内容告一段落了=。=老司机更的速度呢的确是有点慢，忙是一方面，`懒`是另一方面。

![懒](http://upload-images.jianshu.io/upload_images/1835430-eb4d9e47069d8be1.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


不过老司机会把剩下的几个类在下一期说完的=。=

demo老司机放在了网盘里，你可以来[这里](https://pan.baidu.com/s/1mi9COfm)找。

至于镜像控件，老司机封装好了单独放在了一个仓库，你可以来[这里](https://github.com/CodeWicky/Components.git)找。

最后，如果你喜欢老司机的文章，点个关注点个喜欢吧~
