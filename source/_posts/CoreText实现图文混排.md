
---
title: CoreText实现图文混排
layout: post
date: 2016-05-15 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CoreText 
- 图文混排
categories: 图文混排

---

![CoreText实现图文混排](http://upload-images.jianshu.io/upload_images/1835430-07e7d10e9bb52c11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：

- [CoreText实现图文混排](http://www.jianshu.com/p/6db3289fb05d)
- [CoreText实现图文混排之点击事件](http://www.jianshu.com/p/51c47329203e)
- [CoreText实现图文混排之文字环绕及点击算法](http://www.jianshu.com/p/e154047b0f98)
- - -
也好久没来写博客了，主要是最近也工作了，手头的事有点多，一时间也就断了，闲下来了我就来补博客了，刚好最近也做了很多东西，放在这里也算给自己做个笔记吧。

----------
# CoreText
最近公司做了一个项目，需要用到`图文混排`技术。于是呢就疯狂地在网上搜刮资料。  
不过很不幸的是，百度的CoreText资料还是比较少滴，翻来覆去就那几个版本。  
然而我又上不去谷歌，so，困难重重啊。  
不过虽然资料少，不够前辈们给的贡献终于还是在我的努力下都被我消化了，然后我也来做个笔记。

- - - - 
## CoreText的介绍
Core Text 是基于 iOS 3.2+ 和 OSX 10.5+ 的一种能够对文本格式和文本布局进行精细控制的文本引擎。  
它良好的结合了 UIKit 和 Core Graphics/Quartz：  


	UIKit 的 UILabel 允许你通过在 IB 中简单的拖曳添加文本，但你不能改变文本的颜色和其中的单词。  
	Core Graphics/Quartz几乎允许你做任何系统允许的事情，但你需要为每个字形计算位置，并画在屏幕上。
	Core Text 正结合了这两者！你可以完全控制位置、布局、类似文本大小和颜色这样的属性，而 Core Text 将帮你完善其它的东西——类似文本换行、字体呈现等等。

以上就是对CoreText的介绍。

<!-- more -->


---------
## 老司机对CoreText实现图文混排的一些理解

老司机认为，图文混排中使用到的CoreText只是CoreText庞大体系中一个对富文本的**增强**的一部分。
我个人想法啊，我读书少，理解的可能不到位，不过你咬我啊。

恩，我又逗逼了一波，说好的大师气质呢，下面开始严肃了啊。

严肃的就是iOS7新推出的类库`Textkit`，其实是在之前推出的`CoreText上的封装`，根据苹果的说法，他们开发了两年多才完成，而且他们在开发时候也将表情混排作为一个使用案例进行研究，所以要`实现表情混排将会非常容易`。

苹果引入TextKit的目的并非要取代已有的CoreText框架，虽然`CoreText的主要作用也是用于文字的排版和渲染`，但它是一种先进而又处于底层技术，如果我们需要将文本内容直接渲染到图形上下文(Graphics context)时，`从性能和易用性来考虑`，`最佳方案就是使用CoreText`。
原理的东西学一学总没有坏处。因此，还是有必要去学一学CoreText的。  
那我们开始学习吧。
- - - - -
# 富文本
老司机说过，我要讲的只是用来增强富文本的那一部分，那么富文本怎么使用呢。

富文本是什么呢？

> 富文本格式(RTF)规范是为了便于在应用程序之间轻松转储格式化文本和图形的一种编码方法。
现在，用户可以利用特定转换软件，在不同系统如MS-DOS、Windows、OS/2、Macintosh和Power Macintosh的应用程序之间转移字处理文档。
RTF规范提供一种在不同的输出设备、操作环境和操作系统之间交换文本和图形的一种格式。
RTF使用ANSI, PC-8, Macintosh, 或IBM PC字符集控制文档的表示法和格式化，包括屏幕显示和打印
。凭借RTF规范，不同的操作系统和不同的软件程序创建的文档能够在这些操作系统和应用程序之间传递。
将一个格式化的文件转换为RTF文件的软件称为RTF书写器。
RTF书写器用于分离现有文本中的程序控制信息，并且生成一个包含文本和与之相关的RTF组的新文件。将RTF文件转换成格式化文件的软件则称为RTF阅读器。

简单的说，附`带有每一个文字属性的字符串`，就是富文本。  
在iOS中，我们有一个专门的类来处理富文本 `AttributeString`。  
- - - 
## 富文本的基本使用方法
  
诶，标题越来越小了，都4个#号了，说明我扯远了啊。不过要想使用CoreText不会富文本还是不行啊。

来吧。  
AttributedString也分为`NSAttributedString`和`NSMutableAttributedString`两个类，类似于String，我就不赘述了。  
富文本本质上没有什么难度，只要给指定的字符串附上指定的属性就好了。下面给出富文本的一些基本方法。  

- -initWithString:`以NSString初始化一个富文本对象`  
- -setAttributes:range:`为富文本中的一段范围添加一些属性`，第一个参数是个NSDictionary字典，第二个参数是NSRange。
- -addAttribute:value:range:`添加一个属性`
- -addAttributes:range:`添加多个属性`
- -removeAttribute:range:`移除属性`
额,老司机知道这么说不直观，来来来，上代码。  

```
NSDictionary * dic = @{NSFontAttributeName:[UIFont fontWithName:@"Zapfino" size:20],NSForegroundColorAttributeName:[UIColor redColor],NSUnderlineStyleAttributeName:@(NSUnderlineStyleSingle)};
    NSMutableAttributedString * attributeStr = [[NSMutableAttributedString alloc] initWithString:@"0我是一个富文本，9听说我有很多属性，19I will try。32这里清除属性."];
//    设置属性
    [attributeStr setAttributes:dic range:NSMakeRange(0, attributeStr.length)];
//    添加属性
    [attributeStr addAttribute:NSFontAttributeName value:[UIFont systemFontOfSize:30] range:NSMakeRange(9, 10)];
    [attributeStr addAttribute:NSForegroundColorAttributeName value:[UIColor cyanColor] range:NSMakeRange(13, 13)];
//    添加多个属性
    NSDictionary * dicAdd = @{NSBackgroundColorAttributeName:[UIColor yellowColor],NSLigatureAttributeName:@1};
    [attributeStr addAttributes:dicAdd range:NSMakeRange(19, 13)];
//    移除属性
    [attributeStr removeAttribute:NSFontAttributeName range:NSMakeRange(32, 9)];
    UILabel * label = [[UILabel alloc] initWithFrame:CGRectMake(100, 100, 200, 400)];
    label.numberOfLines = 0;
    label.attributedText = attributeStr;
```

这里你要注意一下，给label的一定是给他的`attributedText`属性，你给text是不行的。   
是不是用起来很简单，富文本，跟字典没什么区别么。

----
# CoreText绘制富文本

是不是终于进入正题了。其实之所以说那么多，还是为了你看完就能保证会用啊，否则你不会富文本你自己还要查找富文本相关资料。

**Come On！**

CoreText实现图文混排其实就是在富文本中`插入一个空白的图片占位符`的富文本字符串，通过`代理设置相关的图片尺寸信息`，根据从富文本得到的frame`计算图片绘制的frame再绘制图片`这么一个过程。  

先来整体代码

## 全部代码

```
-(void)drawRect:(CGRect)rect
{
    [super drawRect:rect];
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetTextMatrix(context, CGAffineTransformIdentity);    CGContextTranslateCTM(context, 0, self.bounds.size.height);
    CGContextScaleCTM(context, 1.0, -1.0);
    NSMutableAttributedString * attributeStr = [[NSMutableAttributedString alloc] initWithString:@"\n这里在测试图文混排，\n我是一个富文本"];
    CTRunDelegateCallbacks callBacks;
    memset(&callBacks,0,sizeof(CTRunDelegateCallbacks));
    callBacks.version = kCTRunDelegateVersion1;
    callBacks.getAscent = ascentCallBacks;
    callBacks.getDescent = descentCallBacks;
    callBacks.getWidth = widthCallBacks;
    NSDictionary * dicPic = @{@"height":@129,@"width":@400};
    CTRunDelegateRef delegate = CTRunDelegateCreate(& callBacks, (__bridge void *)dicPic);
    unichar placeHolder = 0xFFFC;
    NSString * placeHolderStr = [NSString stringWithCharacters:&placeHolder length:1];
    NSMutableAttributedString * placeHolderAttrStr = [[NSMutableAttributedString alloc] initWithString:placeHolderStr];
    CFAttributedStringSetAttribute((CFMutableAttributedStringRef)placeHolderAttrStr, CFRangeMake(0, 1), kCTRunDelegateAttributeName, delegate);
    CFRelease(delegate);
    [attributeStr insertAttributedString:placeHolderAttrStr atIndex:12];
    CTFramesetterRef frameSetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributeStr);
    CGMutablePathRef path = CGPathCreateMutable();
    CGPathAddRect(path, NULL, self.bounds);
    NSInteger length = attributeStr.length;
    CTFrameRef frame = CTFramesetterCreateFrame(frameSetter, CFRangeMake(0, length), path, NULL);
    CTFrameDraw(frame, context);
    
    UIImage * image = [UIImage imageNamed:@"bd_logo1"];
    CGRect imgFrm = [self calculateImageRectWithFrame:frame];
    CGContextDrawImage(context,imgFrm, image.CGImage);
    CFRelease(frame);
    CFRelease(path);
    CFRelease(frameSetter);
}
static CGFloat ascentCallBacks(void * ref)
{
    return [(NSNumber *)[(__bridge NSDictionary *)ref valueForKey:@"height"] floatValue];
}
static CGFloat descentCallBacks(void * ref)
{
    return 0;
}
static CGFloat widthCallBacks(void * ref)
{
    return [(NSNumber *)[(__bridge NSDictionary *)ref valueForKey:@"width"] floatValue];
}


-(CGRect)calculateImageRectWithFrame:(CTFrameRef)frame
{
    NSArray * arrLines = (NSArray *)CTFrameGetLines(frame);
    NSInteger count = [arrLines count];
    CGPoint points[count];
    CTFrameGetLineOrigins(frame, CFRangeMake(0, 0), points);
    for (int i = 0; i < count; i ++) {
        CTLineRef line = (__bridge CTLineRef)arrLines[i];
        NSArray * arrGlyphRun = (NSArray *)CTLineGetGlyphRuns(line);
        for (int j = 0; j < arrGlyphRun.count; j ++) {
            CTRunRef run = (__bridge CTRunRef)arrGlyphRun[j];
            NSDictionary * attributes = (NSDictionary *)CTRunGetAttributes(run);            CTRunDelegateRef delegate = (__bridge CTRunDelegateRef)[attributes valueForKey:(id)kCTRunDelegateAttributeName];
            if (delegate == nil) {
                continue;
            }
            NSDictionary * dic = CTRunDelegateGetRefCon(delegate);
            if (![dic isKindOfClass:[NSDictionary class]]) {
                continue;
            }
            CGPoint point = points[i];
            CGFloat ascent;
            CGFloat descent;
            CGRect boundsRun;
            boundsRun.size.width = CTRunGetTypographicBounds(run, CFRangeMake(0, 0), &ascent, &descent, NULL);
            boundsRun.size.height = ascent + descent;
            CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringRange(run).location, NULL);
            boundsRun.origin.x = point.x + xOffset;
            boundsRun.origin.y = point.y - descent;
            CGPathRef path = CTFrameGetPath(frame);
            CGRect colRect = CGPathGetBoundingBox(path);
            CGRect imageBounds = CGRectOffset(boundsRun, colRect.origin.x, colRect.origin.y);
            return imageBounds;
        }
    }
    return CGRectZero;
}
```

不瞒你说，我看着代码都烦，也怕，所以放心，老司机会一句一句给你解释的。
- - - 

## 分段解析
### 准备工作

```
CGContextRef context = UIGraphicsGetCurrentContext();
CGContextSetTextMatrix(context, CGAffineTransformIdentity);    
CGContextTranslateCTM(context, 0, self.bounds.size.height);
CGContextScaleCTM(context, 1.0, -1.0);
```
先要来一个背景介绍哈

    /*
     coreText 起初是为OSX设计的，而OSX得坐标原点是左下角，y轴正方向朝上。iOS中坐标原点是左上角，y轴正方向向下。
     若不进行坐标转换，则文字从下开始，还是倒着的
     	如下图(盗的图，别打我)
     */
     
![系统坐标系](http://upload-images.jianshu.io/upload_images/1835430-597db8c1833bf0c1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  


这四句什么意思呢？  

#### 获取上下文  
```
CGContextRef context = UIGraphicsGetCurrentContext();//获取当前绘制上下文
```

  
为什么要回去上下文呢？因为我们所有的绘制操作都是在上下文上进行绘制的。 
 
#### 坐标转换

```
CGContextSetTextMatrix(context, CGAffineTransformIdentity);//设置字形的变换矩阵为不做图形变换  
    CGContextTranslateCTM(context, 0, self.bounds.size.height);//平移方法，将画布向上平移一个屏幕高  
    CGContextScaleCTM(context, 1.0, -1.0);//缩放方法，x轴缩放系数为1，则不变，y轴缩放系数为-1，则相当于以x轴为轴旋转180度
```

正如之上的背景说的，coreText使用的是`系统坐标`，然而我们平时所接触的iOS的都是`屏幕坐标`，所以要将屏幕坐标系`转换`系统坐标系，这样才能与我们想想的坐标互相对应。  
`事实上呢，这三句是翻转画布的固定写法，这三句你以后会经常看到的。`

继续。
- - -
#### 图片的代理的设置

> 事实上，图文混排就是在要插入图片的位置插入一个富文本类型的占位符。通过CTRUNDelegate设置图片

```
NSMutableAttributedString * attributeStr = [[NSMutableAttributedString alloc] initWithString:@"\n这里在测试图文混排，\n我是一个富文本"];//这句不用我多说吧，最起码得有个富文本啊才能插入不是。
/*
 设置一个回调结构体，告诉代理该回调那些方法
 */
CTRunDelegateCallbacks callBacks;//创建一个回调结构体，设置相关参数
memset(&callBacks,0,sizeof(CTRunDelegateCallbacks));//memset将已开辟内存空间 callbacks 的首 n 个字节的值设为值 0, 相当于对CTRunDelegateCallbacks内存空间初始化
callBacks.version = kCTRunDelegateVersion1;//设置回调版本，默认这个
callBacks.getAscent = ascentCallBacks;//设置图片顶部距离基线的距离
callBacks.getDescent = descentCallBacks;//设置图片底部距离基线的距离
callBacks.getWidth = widthCallBacks;//设置图片宽度
```

**注意了，这里经GreyLove提醒有重要改动，就是这里，添加了memset，码字的时候少忘码了一句话。怪我粗心，十分抱歉，我会通知每一个留言的同学。**


为什么要设置一个回调结构体呢？
因为coreText中大量的调用c的方法。`事实上你会发现大部分跟系统底层有关的都需要调c的方法。`所以设置代理要按照人家的方法来啊。

看看这几句代码也很好懂，就是注释中写的意思。  
后三句分别就是说当我需要走这些代理的时候都会走那些`代理方法`。  
好吧，扯到这又要补充知识了。这个距离什么东西呢？
![字形](http://upload-images.jianshu.io/upload_images/1835430-945506c609eeac5d.gif?imageMogr2/auto-orient/strip)

对对，这呢就是一个CTRun的尺寸图，什么你问CTRun是啥？还没到那呢，后面会详细介绍。
在这你只要知道，一会我们绘制图片的时候实际上实在一个CTRun中绘制这个图片，那么CTRun绘制的坐标系中，他会以`origin点作为原点`进行绘制。  
`基线为过原点的x轴`，`ascent即为CTRun顶线距基线的距离`，`descent即为底线距基线的距离`。  
我们绘制图片应该从原点开始绘制，图片的高度及宽度及CTRun的高度及宽度，我们`通过代理设置CTRun的尺寸间接设置图片的尺寸。`

---

```
/*
 创建一个代理
*/
    NSDictionary * dicPic = @{@"height":@129,@"width":@400};//创建一个图片尺寸的字典，初始化代理对象需要
    CTRunDelegateRef delegate = CTRunDelegateCreate(& callBacks, (__bridge void *)dicPic);//创建代理
```

上面只是设置了回调结构体，然而我们还没有告诉这个代理我们要的`图片尺寸`。
所以这句话就在`设置代理的时候绑定了一个返回图片尺寸的字典`。  
事实上此处你可以`绑定任意对象`。此处你`绑定的对象既是回调方法中的参数ref`。
  
好吧就然说到这我就直接把那三个回调方法说了吧，放在一起比较好理解一些。 
 
```
static CGFloat ascentCallBacks(void * ref)
{
    return [(NSNumber *)[(__bridge NSDictionary *)ref valueForKey:@"height"] floatValue];
}
static CGFloat descentCallBacks(void * ref)
{
    return 0;
}
static CGFloat widthCallBacks(void * ref)
{
    return [(NSNumber *)[(__bridge NSDictionary *)ref valueForKey:@"width"] floatValue];
}

```

上文说过，ref既是创建代理是绑定的对象。所以我们在这里，`从字典中分别取出图片的宽和高`。

值得注意的是，由于是c的方法，所以也没有什么对象的概念。是一个`指针类型的数据`。不过oc的对象其实也就是c的结构体。我们可以通过类型转换获得oc中的字典。  
`__bridge既是C的结构体转换成OC对象时需要的一个修饰词`。

老司机敲字慢啊，敲到这都两个小时了，容我喝口水。  

你们喝过红色的尖叫么？老司机喝了那种烟头泡的水之后`精神满满`的继续敲字。（那水超难喝，你可以挑战一下）  
诶，说好的严肃呢？

---
#### 图片的插入
首先创建一个富文本类型的图片占位符，绑定我们的代理

```
unichar placeHolder = 0xFFFC;//创建空白字符
    NSString * placeHolderStr = [NSString stringWithCharacters:&placeHolder length:1];//已空白字符生成字符串
    NSMutableAttributedString * placeHolderAttrStr = [[NSMutableAttributedString alloc] initWithString:placeHolderStr];//用字符串初始化占位符的富文本
    CFAttributedStringSetAttribute((CFMutableAttributedStringRef)placeHolderAttrStr, CFRangeMake(0, 1), kCTRunDelegateAttributeName, delegate);//给字符串中的范围中字符串设置代理
    CFRelease(delegate);//释放（__bridge进行C与OC数据类型的转换，C为非ARC，需要手动管理）
```
这里富文本的知识上文中已经介绍过了。不过老司机猜你有三个疑问。  

* 这个添加属性的方法怎么是这个样子的？  
因为这里是添加CTRunDelegate这种数据类型，要用CoreText专门的方法，不过其实就是形式不同，作用一样的。
* 为什么这里富文本类型转换的时候不用_bridge呢？老司机你不是说需要修饰词么？你是不是骗我？(markDown语法冲突我少打一个下划线)  
真没有，事实上不是所有数据转换的时候都需要__bridge。你要问我怎么区分？那好我告诉你，C中就是传递指针的数据就不用。比如说字符串，数组。原因老司机现在解释不通，等我能组织好语言的。
* 为什么还要释放？我是ARC环境啊  
不好意思，我也是。不过为什么要释放呢？因为你进行了类型转换之后就不属于对象了，也不再归自动引用计数机制管理了，所以你得`手动管理`咯。
- - -

然后将占位符插入到我们的富文本中

```
[attributeStr insertAttributedString:placeHolderAttrStr atIndex:12];//将占位符插入原富文本
```
此处我就不赘述了，富文本的知识你只要类比字典就好了。  
至此，我们已经生成好了我们要的带有图片信息的富文本了，接下来我们只要在画布上`绘制`出来这个富文本就好了。
- - -
#### 绘制
绘制呢，又分成两部分，`绘制文本`和`绘制图片`。你问我为什么还分成两个？

因为富文本中你添加的图片只是一个`带有图片尺寸的空白占位符`啊，你绘制的时候他只会绘制出相应尺寸的空白占位符，所以什么也显示不了啊。  
那怎么显示图片啊？拿到占位符的坐标，`在占位符的地方绘制相应大小的图片`就好了。恩，说到这，图文混排的原理已经说完了。

先来绘制文本吧。
- - -
##### 绘制文本

```
CTFramesetterRef frameSetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributeStr);//一个frame的工厂，负责生成frame
CGMutablePathRef path = CGPathCreateMutable();//创建绘制区域
CGPathAddRect(path, NULL, self.bounds);//添加绘制尺寸
NSInteger length = attributeStr.length;
CTFrameRef frame = CTFramesetterCreateFrame(frameSetter, CFRangeMake(0,length), path, NULL);//工厂根据绘制区域及富文本（可选范围，多次设置）设置frame
CTFrameDraw(frame, context);//根据frame绘制文字
```
frameSetter是根据富文本生成的一个frame生成的工厂，你可以`通过framesetter`以及你想要绘制的`富文本的范`围获取`该CTRun的frame`。  
但是你需要注意的是，获取的frame是`仅绘制你所需要的那部分富文本的frame`。即当前情况下，你绘制范围定为（10，1），那么你得到的尺寸是只绘制（10，1）的尺寸，他应该从屏幕左上角开始（因为你改变了坐标系），而`不是当你绘制全部富文本时他该在的位置`。

然后建立一会绘制的尺寸，实际上就是在指定你的`绘制范围`。  
接着生成`整个富文本绘制所需要的frame`。因为范围是全部文本，所以获取的frame即为全部文本的frame(此处老司机希望你一定要搞清楚全部与指定范围获取的frame他们都是从左上角开始的，否则你会进入一个奇怪的误区，稍后会提到的)。  
最后，根据你获得的frame，绘制`全部富文本`。
- - -
##### 绘制图片
上面你已经绘制出文字，不过没有图片哦，接下来绘制图片。
绘制图片用下面这个方法，通用的哦  

```
CGContextDrawImage(context,imgFrm, image.CGImage);//绘制图片
```
我们可以看到这个方法有三个参数，分别是`context，frame，以及image`。
要什么就给他什么好咯，context和image都好说，context就是当前的上下文，最开始获得那个。image就是你要添加的那个图片，不过是`CGImage类型`。通过UIImage转出CGImage就好了，我们重点讲一下frame的获取。  

###### frame的获取
记得我之前说的误区么？这里我们要获得Image的frame，你有没有想过我们的frameSetter？

我也想过，不过就像我说的，你单独用frameSetter求出的image的frame是不正确的，那是只绘制image而得的坐标，所以哪种方法不能用哦，要用下面的方法。    

你们一定发现，我获取frame的方法单独写了一个方法，为什么呢？  
1.将`代码分离`，方便修改。  
2.最主要的是这部分代码到哪里都能用，达到`复用`效果。  

```
NSArray * arrLines = (NSArray *)CTFrameGetLines(frame);//根据frame获取需要绘制的线的数组
NSInteger count = [arrLines count];//获取线的数量
CGPoint points[count];//建立起点的数组（cgpoint类型为结构体，故用C语言的数组）
CTFrameGetLineOrigins(frame, CFRangeMake(0, 0), points);//获取起点
```
第一句呢，获取绘制frame中的所有CTLine。CTLine，又不知道了吧，老司机又要无耻的盗图了。  
![CTFrame组成](http://upload-images.jianshu.io/upload_images/1835430-823846a97a7b220e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
上面呢，我们能看到一个CTFrame绘制的原理。

* CTLine 可以看做Core Text绘制中的一行的对象 通过它可以获得当前行的line ascent,line descent ,line leading,还可以获得Line下的所有Glyph Runs
* CTRun 或者叫做 Glyph Run，是一组共享想相同attributes（属性）的字形的集合体

一个CTFrame有几个CTLine组成，有几行文字就有几行CTLine。一个CTLine有包含多个CTRun，一个CTRun是所有属性都相同的那部分富文本的绘制单元。所以CTRun是CTFrame的`基本绘制单元`。  
接着说我们的代码。  
为什么我获取的数组需要进行类型转换呢？因为CTFrameGetLines（）返回值是`CFArrayRef类型`的数据。就是一个c的数组类型吧，暂且先这么理解，所以需要转换。  

那为什么不用__bridge呢？记得么，我说过，`本身就传地址的数据是不用桥接的`。就是这样。  
然后获取数组的元素个数。有什么用呢，因为我们要用到每个CTLine的`原点坐标`进行计算。每个CTLine都有自己的origin。所以要生成一个`相同元素个数的数组去盛放origin对象`。  
然后用CTFrameGetLineOrigins获取所有原点。  
到此，我们计算frame的准备工作完成了。才完成准备工作。  
- - -
###### 计算frame
思路呢，就是`遍历`我们的frame中的所有CTRun，`检查`他是不是我们绑定图片的那个，如果是，根据该CTRun所在CTLine的origin以及CTRun在CTLine中的横向偏移量`计算`出CTRun的原点，加上其尺寸即为该CTRun的尺寸。

跟绕口令是的，不过就是这么个思路。

```
for (int i = 0; i < count; i ++) {//遍历线的数组
        CTLineRef line = (__bridge CTLineRef)arrLines[i];
        NSArray * arrGlyphRun = (NSArray *)CTLineGetGlyphRuns(line);//获取GlyphRun数组（GlyphRun：高效的字符绘制方案）
        for (int j = 0; j < arrGlyphRun.count; j ++) {//遍历CTRun数组
            CTRunRef run = (__bridge CTRunRef)arrGlyphRun[j];//获取CTRun
            NSDictionary * attributes = (NSDictionary *)CTRunGetAttributes(run);//获取CTRun的属性
            CTRunDelegateRef delegate = (__bridge CTRunDelegateRef)[attributes valueForKey:(id)kCTRunDelegateAttributeName];//获取代理
            if (delegate == nil) {//非空
                continue;
            }
            NSDictionary * dic = CTRunDelegateGetRefCon(delegate);//判断代理字典
            if (![dic isKindOfClass:[NSDictionary class]]) {
                continue;
            }
            CGPoint point = points[i];//获取一个起点
            CGFloat ascent;//获取上距
            CGFloat descent;//获取下距
            CGRect boundsRun;//创建一个frame
            boundsRun.size.width = CTRunGetTypographicBounds(run, CFRangeMake(0, 0), &ascent, &descent, NULL);
            boundsRun.size.height = ascent + descent;//取得高
            CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringRange(run).location, NULL);//获取x偏移量
            boundsRun.origin.x = point.x + xOffset;//point是行起点位置，加上每个字的偏移量得到每个字的x
            boundsRun.origin.y = point.y - descent;//计算原点
            CGPathRef path = CTFrameGetPath(frame);//获取绘制区域
            CGRect colRect = CGPathGetBoundingBox(path);//获取剪裁区域边框
            CGRect imageBounds = CGRectOffset(boundsRun, colRect.origin.x, colRect.origin.y);
            return imageBounds;
```

有了上面的思路这里就很好理解了。  
外层for循环呢，是为了`取到所有的CTLine`。  
类型转换什么的我就不多说了，然后通过CTLineGetGlyphRuns获取一个CTLine中的`所有CTRun`。  
里层for循环是检查每个CTRun。  
通过CTRunGetAttributes拿到该`CTRun的所有属性`。  
通过kvc取得属性中的`代理属性`。  
接下来判断`代理属性是否为空`。因为图片的占位符我们是绑定了代理的，而文字没有。以此区分文字和图片。  
如果代理不为空，通过CTRunDelegateGetRefCon取得生成`代理时绑定的对象`。`判断类型`是否是我们绑定的类型，`防止取得我们之前为其他的富文本绑定过代理`。  
如果两条都符合，ok，这就是`我们要的那个CTRun`。  
开始计算该`CTRun的frame`吧。  
获取`原点`和获取`宽高`被。  
通过`CTRunGetTypographicBounds`取得宽，ascent和descent。有了上面的介绍我们应该知道图片的高度就是ascent+descent了吧。  
接下来获取原点。
CTLineGetOffsetForStringIndex获取对应`CTRun的X偏移量`。
取得对应CTLine的原点的Y，`减去图片的下边距才是图片的原点`，这点应该很好理解。  
至此，我们已经获得了`图片的frame`了。因为只绑定了一个图片，所以直接return就好了，如果多张图片可以继续遍历返回数组。  
获取到图片的frame，我们就可以绘制图片了，用上面介绍的方法。  
- - -
哦，别忘了手动释放你创建的对象哦。  

```
CFRelease(frame);
CFRelease(path);
CFRelease(frameSetter);
```
大功告成。

----
好了，至此你已经完成图片的绘制了。只要在ViewController里面引入你绘制CoreText文本的View正常的初始化添加子视图就可以了。

好吧，这个教程我也是综合了很多资料写出来的。优势是在于我一句一句讲的，几乎每一句都告诉你原理了吧。

恩，我也是在前人的基础上自己总结查阅出来的，难免夹杂着个人理解和部分偏颇，如果各位看官发现我写的有什么不对的地方欢迎与我联系，老司机的邮箱codewicky@163.com。

原谅老司机逗逼的本质，严肃不起来。

下面是一些参考资料：  
[coreText方法，列的很全](http://blog.csdn.net/liliangchw/article/details/8261646)  
[CTRun的详细介绍](http://www.docin.com/p-721844449.html)  
[CTLine的详细介绍](http://blog.csdn.net/majiakun1/article/details/38140411)  
[coreText基本原理及使用方法](http://www.tuicool.com/articles/FfYjim)  
[图文混排的整体介绍](http://my.oschina.net/u/2361492/blog/526814)  
[coreText使用方法](https://my.oschina.net/megan/blog/269042)

你要是喜欢呢，麻烦你动一动你可爱的小手点击一下喜欢或者关注，毕竟老司机这么爱慕虚荣的人，而且老司机会经常更新的。


最后，你问我为什么一直叫自己老司机？哦，因为嘿嘿嘿~~~

哦，最后的最后，若果真有人转载的话，麻烦你注明出处。
[http://www.jianshu.com/p/6db3289fb05d](http://www.jianshu.com/p/6db3289fb05d)
