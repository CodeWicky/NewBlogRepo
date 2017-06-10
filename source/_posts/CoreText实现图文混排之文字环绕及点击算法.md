
---
title: CoreText实现图文混排之文字环绕及点击算法
layout: post
date: 2016-12-25 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CoreText 
- 图文混排
- 点击事件
- 图片环绕 
categories: 图文混排

---

![CoreText实现图文混排之文字环绕及点击算法](http://upload-images.jianshu.io/upload_images/1835430-80b2fc8677bcd42f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系列文章：
- [CoreText实现图文混排](http://www.jianshu.com/p/6db3289fb05d)
- [CoreText实现图文混排之点击事件](http://www.jianshu.com/p/51c47329203e)
- [CoreText实现图文混排之文字环绕及点击算法](http://www.jianshu.com/p/e154047b0f98)

- - -

在这篇文章中你可以看到以下内容：

- 图片环绕的实现方式
- 点击事件获取的优化算法

<!-- more -->


- - -
终于我来完成我CoreText图文混排的最后一章了。

先说一下我为什么会来补发这一章呢？
> 
1.老司机最开始没有留demo，以至于这个博客老司机从发出来到现在整整维护了半年了=。=其实博客里面就是全部代码，但是宝宝们任性的要demo。
>
2.时间长了，阅读量也上去了，老司机觉得自己有必要对粉丝们负责了
>
3.有很多同学询问是否能做出文字环绕的效果，老司机之前的确也没有写过，这一篇是要补上的。
>
4.关于点击事件，老司机在第二篇文章中有提到过一个思路，是每次遍历所有CTRun去做的。后期老司机考虑到遍历的实现效率似乎有些低，所以老司机研究了一下，重新整理思路，优化了一下算法。

基于以上原因，以及一个`阴谋`，老司机又来更文了。

![劳资回来了](http://upload-images.jianshu.io/upload_images/1835430-6e1368fd36b6e027.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)






看了本篇博客，老司机能够帮你实现如下效果


![CoreText](http://upload-images.jianshu.io/upload_images/1835430-e44f7e218a059e4b.gif?imageMogr2/auto-orient/strip)


这篇博客是以前两篇博客作为知识铺垫的，如果没有看过前两篇博客的童靴建议你去[补票](http://www.jianshu.com/p/6db3289fb05d)。当然本身你就了解CoreText相关知识的话也可以直接看本篇文章。



- - -
### 全部代码
优化算法以后，代码有些许改变，不过主体思路是一致的。下面是全部代码。

```
@interface CoreTextV ()
{
    CTFrameRef _frame;
    NSInteger _length;
    CGRect _imgFrm;
    NSMutableArray * arrText;
}
@end
@implementation CoreTextV

-(void)drawRect:(CGRect)rect
{
    [super drawRect:rect];
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetTextMatrix(context, CGAffineTransformIdentity);    CGContextTranslateCTM(context, 0, self.bounds.size.height);
    CGContextScaleCTM(context, 1.0, -1.0);
    arrText = [NSMutableArray array];
    NSMutableAttributedString * attributeStr = [[NSMutableAttributedString alloc] initWithString:@"123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890"];
    [attributeStr addAttribute:NSForegroundColorAttributeName value:[UIColor whiteColor] range:NSMakeRange(0, attributeStr.length)];
    CTRunDelegateCallbacks callBacks;
    memset(&callBacks, 0, sizeof(CTRunDelegateCallbacks));
    callBacks.version = kCTRunDelegateVersion1;    	 callBacks.getAscent = ascentCallBacks;    	callBacks.getDescent = descentCallBacks;    	callBacks.getWidth = widthCallBacks;
    NSDictionary * dicPic = @{@"height":@90,@"width":@160};
    CTRunDelegateRef delegate = CTRunDelegateCreate(& callBacks, (__bridge void *)dicPic);
    unichar placeHolder = 0xFFFC;
    NSString * placeHolderStr = [NSString stringWithCharacters:&placeHolder length:1];
    NSMutableAttributedString * placeHolderAttrStr = 	[[NSMutableAttributedString alloc] initWithString:placeHolderStr];
    CFAttributedStringSetAttribute((CFMutableAttributedStringRef)placeHolderAttrStr, CFRangeMake(0, 1), kCTRunDelegateAttributeName, delegate);    CFRelease(delegate);
    [attributeStr insertAttributedString:placeHolderAttrStr atIndex:300];
    NSDictionary * activeAttr = @{NSForegroundColorAttributeName:[UIColor redColor],@"click":NSStringFromSelector(@selector(click))};
    [attributeStr addAttributes:activeAttr range:NSMakeRange(100, 30)];
    
    [attributeStr addAttributes:activeAttr range:NSMakeRange(400, 100)];

    CTFramesetterRef frameSetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributeStr);
    UIBezierPath * path = [UIBezierPath bezierPathWithRect:self.bounds];
    UIBezierPath * cirP = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(100, 100, 100, 200)];
    [path appendPath:cirP];
    _length = attributeStr.length;
    _frame = CTFramesetterCreateFrame(frameSetter, CFRangeMake(0, _length), path.CGPath, NULL);
    CTFrameDraw(_frame, context);
    
    UIImage * image = [UIImage imageNamed:@"1.jpg"];
    [self handleActiveRectWithFrame:_frame];
    CGContextDrawImage(context,_imgFrm, image.CGImage);
    
    CGContextDrawImage(context, cirP.bounds, [[UIImage imageNamed:@"1.jpg"] dw_ClipImageWithPath:cirP mode:(DWContentModeScaleAspectFill)].CGImage);
    CFRelease(_frame);
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


-(void)handleActiveRectWithFrame:(CTFrameRef)frame
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
            NSDictionary * attributes = (NSDictionary *)CTRunGetAttributes(run);
            CTRunDelegateRef delegate = (__bridge CTRunDelegateRef)[attributes valueForKey:(id)kCTRunDelegateAttributeName];
            CGPoint point = points[i];
            if (delegate == nil) {
                NSString * string = attributes[@"click"];
                if (string) {
                    [arrText addObject:[NSValue valueWithCGRect:[self getLocWithFrame:frame CTLine:line CTRun:run origin:point]]];
                }
                continue;
            }
            NSDictionary * metaDic = CTRunDelegateGetRefCon(delegate);
            if (![metaDic isKindOfClass:[NSDictionary class]]) {
                continue;
            }
            _imgFrm = [self getLocWithFrame:frame CTLine:line CTRun:run origin:point];
        }
    }
}

-(CGRect)getLocWithFrame:(CTFrameRef)frame CTLine:(CTLineRef)line CTRun:(CTRunRef)run origin:(CGPoint)origin
{
    CGFloat ascent;
    CGFloat descent;
    CGRect boundsRun;
    boundsRun.size.width = CTRunGetTypographicBounds(run, CFRangeMake(0, 0), &ascent, &descent, NULL);
    boundsRun.size.height = ascent + descent;
    CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringRange(run).location, NULL);
    boundsRun.origin.x = origin.x + xOffset; 
    boundsRun.origin.y = origin.y - descent;
    CGPathRef path = CTFrameGetPath(frame);
    CGRect colRect = CGPathGetBoundingBox(path); 
    CGRect deleteBounds = CGRectOffset(boundsRun, colRect.origin.x, colRect.origin.y);
    return deleteBounds;
}

-(CGRect)convertRectFromLoc:(CGRect)rect
{
    return CGRectMake(rect.origin.x, self.bounds.size.height - rect.origin.y - rect.size.height, rect.size.width, rect.size.height);
}

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch * touch = [touches anyObject];
    CGPoint location = [touch locationInView:self];
    CGRect imageFrmToScreen = [self convertRectFromLoc:_imgFrm];
    if (CGRectContainsPoint(imageFrmToScreen, location)) {
        [[[UIAlertView alloc] initWithTitle:nil message:@"你点击了图片" delegate:nil cancelButtonTitle:@"好的" otherButtonTitles:nil] show];
        return;
    }
    [arrText enumerateObjectsUsingBlock:^(NSValue * rectV, NSUInteger idx, BOOL * _Nonnull stop) {
        CGRect textFrmToScreen = [self convertRectFromLoc:[rectV CGRectValue]];
        if (CGRectContainsPoint(textFrmToScreen, location)) {
            [self click];
            *stop = YES;
        }
    }];
}

-(void)click
{
    [[[UIAlertView alloc] initWithTitle:nil message:@"你点击了文字" delegate:nil cancelButtonTitle:@"好的" otherButtonTitles:nil] show];
}
@end

```

只关心结果或者着急写项目的童靴看到这里就足够了，因为所有代码都在，想找demo的话就去文章末尾找吧。因为接下来老司机要`开始扯淡了`。。。跟你们讲讲一切的`实现思路`。
- - -
###图片环绕的实现方式

由于我只是给个demo，所以一切代码均从简写。`实际过程中，代码应进行封装分块。`

我们将视线集中到`drawRect`方法中吧。

之前的文章老司机讲过，我们在drawRect中绘制文本的时候主要是`根据Path去绘制`的。

```
UIBezierPath * path = [UIBezierPath bezierPathWithRect:self.bounds];
UIBezierPath * cirP = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(100, 100, 100, 200)];
[path appendPath:cirP];
_length = attributeStr.length;
_frame = CTFramesetterCreateFrame(frameSetter, CFRangeMake(0, _length), path.CGPath, NULL);
CTFrameDraw(_frame, context);
```

我们可以看到，我们是以path和frameSetter去生成我们绘制文本的frame的。所以说，只要在这个地方我们`传入的path中将特殊区域排除`我们获得的`frame就不包含该区域`，从而`绘制的文本也不会在该区域中绘制`。

所以说上述的代码你看到的应该是这样子的文字区域


![排除文字区域](http://upload-images.jianshu.io/upload_images/1835430-b9e2009892b5bff5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里你可能会有个疑问，问什么我cirP的rect是CGRectMake(100, 100, 100, 200)，这个排除的区域却在那里。这里你还记得老司机在第一篇文章里就说过`屏幕坐标系统跟系统坐标系统的区别`呢，原因就在这。

也就是说，到了这里，我们只要绘制出这个椭圆形的图片就可以了。这你可能需要借助老司机之前写好的工具类，在这个仓库里的[DWImageUtils](https://github.com/CodeWicky/-Tools)就是了。如果好用记得给我个**`star`**吧。

有了这个工具类，你就可以这样生成`椭圆图片`了
> [image dw_ClipImageWithPath:cirP mode:(DWContentModeScaleAspectFill)]
> 

有了图片了，情况基本就变成了我们熟悉的状况了，绘制图片

>  CGContextDrawImage(context, cirP.bounds, [[UIImage imageNamed:@"1.jpg"] dw_ClipImageWithPath:cirP mode:(DWContentModeScaleAspectFill)].CGImage);
> 

至此，我们就绘制出环绕的文本了。也算真正的实现所谓的图文混排了。

- - - 
### 点击事件获取的优化算法

首先老司机来讲一下目前老司机了解到的几种获取点击事件的方式。

#### CTLineGetStringIndexForPosition

主流方式就是当前大部分基于CoreText封装的富文本展示类（包括`TTTAttributedLabel`、`NIAttributedLabel`和`FTCoreTextView`）中使用的方法 `CTLineGetStringIndexForPosition`。这个方法是`获取当前点在所在文字处于当前绘制文本的索引值`。事实上如果没有一些其他因素的话，能使用这个方法是最简便快捷的。然而老司机为什么没有使用这个方法去获取点击事件呢？请看下面的动图👇


![CTLineGetStringIndexForPosition](http://upload-images.jianshu.io/upload_images/1835430-d6a8b77cf8ee04fd.gif?imageMogr2/auto-orient/strip)


这里老司机是以TTTAttributedLabel为样本做了一个点击事件的Demo。

先明确一点，有下划线的区域应该为实际点击响应区域。可以看到，`实际的响应区域相比预期响应区域x坐标会整体向左偏移一定区域`。

实际使用中CTLineGetStringIndexForPosition这个方法获取一个字的index范围是`这个字前面大概半个字开始到这个字中间的位置`。从这个字的中间到这个字的后半个字就会获得下一个字的index。举个例子：（勾选的勾字index应该为0，当点击勾字左半部分的时候返回0，右半部分返回1）。

老司机查阅了很多资料，有的资料说这个方法在当有段前缩进或者首行缩进的时候，并不准确，不会跟着缩进而进行偏移。然而老司机在将段前缩进设为0仍然有这个问题。老司机也不知道具体问题在哪，然而老司机有强迫症的不能允许这`半个字的误差`，所以老司机当时决定不用这个方法，自己另辟蹊径。就有了老司机当时的遍历每个CTRun的算法。

多说一句，CTLineGetStringIndexForPosition这个方法还有另一个作用还是很好用的。`这个方法最好的用处就是判断一行CTLine最多容纳多少的字符，只需把这个point的x位置调很大（超过CTFrame path的宽度）就可以了`。

#### 遍历CTRun比较法

老司机当时觉得半个字的误差实在是难以容忍，所以老司机舍近求远想出了这套`遍历CTRun`的算法。因为执行效率上一个屏幕内能展示的文字所包含的CTRun的数量在遍历过程中并不会造成过多的性能浪费，所以老司机当时也没有在意。

直到后来老司机的项目中由于要尽量少的使用三方SDK，所以自告奋勇的把自己写的coreText的可点击label引入到工程里面。然而项目经理看了源码后表示虽然他没用过CoreText，但是遍历真的很蛋疼，决定引入一个TTT。老司机的心情瞬间跌入谷底。`老司机当即决定，我要优化算法`。

#### 计算并缓存法

之所以说优化算法，没有说不用遍历是因为CoreText就那么些东西，获取图片的_frame还是需要遍历整个CTFrame中的所有CTRun的。所以老司机花了整整一个礼拜也`没找到替代遍历的方法`。

终于有一天，感受到了月亮的召唤，老司机`变！身！了！`


![变身了](http://upload-images.jianshu.io/upload_images/1835430-23cd1c8301471fbe.gif?imageMogr2/auto-orient/strip)


顺便想到了一个思路，`避免不了遍历我就只遍历一次就好了`。一次遍历中拿到所有活动图片和活动文字的frame，然后事情就简单多了，按照点击图片的处理方式处理文字就好了。

所以老司机就想了一个办法期望在遍历的时候可以拿到活动文字的特征点，从而获取活动范围。老司机顺理成章的就`给想要添加点击事件的活动文本加了click这么一个属性`。（demo中老司机就随便写了，实际要慎重考虑叫什么名字不会被觉得太Low😂）

其实实现上的思路很简单，只是之前没想到，`感谢月亮的召唤吧`还是。


![关我毛事](http://upload-images.jianshu.io/upload_images/1835430-fc2c885752867d23.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
NSDictionary * activeAttr = @{NSForegroundColorAttributeName:[UIColor redColor],@"click":NSStringFromSelector(@selector(click))};
[attributeStr addAttributes:activeAttr range:NSMakeRange(100, 30)];
[attributeStr addAttributes:activeAttr range:NSMakeRange(400, 100)];
```
这里老司机很随意的添加了一个click属性。

```
[arrText addObject:[NSValue valueWithCGRect:[self getLocWithFrame:frame CTLine:line CTRun:run origin:point]]];
```
然后这里老司机就把活动文本的frame计算出来了。

思路就是这么简单。本着一年内保证售后服务的原则，老司机会给在前两篇博客中要demo的童鞋再发一份最新的demo通知新算法。

另外老司机这里要提醒你一点的就是，`文字frame不同于获取图片的frame`。由于`图片`是在一个空白占位符上绘制文字，`所以一定是以一个CTRun进行绘制的`。但是第一篇文章中老司机说过，`每个CTRun是所有具有相同属性的连续同行文字的集合`。针对CTRun的特性，我们不难想到，文字由于可能出现两行，也有可能会活动文本的字体字号等其他属性不尽相同导致`一段文字由两个CTRun进行绘制`，所以不能单纯的保存一个frame，而是`要以一个数组容纳他`。再通过一些逻辑将不同的活动文本区别开来。由于是demo所以一切从简老司机没有说怎么区分活动文本，给个思路，就是你给click属性`绑定的value就可以用作区分`（这句好拗口）。至此，全部的文字点击及图片点击事件的大体思路讲述完毕。

- - -
为了吸取教训，这次老司机会留下demo。恩，也要去前两篇通知他们第三篇有demo。
### [去这里下载demo](https://pan.baidu.com/s/1skQN7Md)

然后，

广告时间了：

痛定思痛的老司机写了一个真正支持图文混排点击事件的Label控件。你可以实现如下的效果。


![DWCoreTextLabel](http://upload-images.jianshu.io/upload_images/1835430-bc004dc959f7b675.gif?imageMogr2/auto-orient/strip)


插入图片、绘制图片、添加事件统统一句话实现~


![一句话实现](http://upload-images.jianshu.io/upload_images/1835430-456beaed3f710b02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


尽可能保持系统Label属性让你可以无缝过渡使用~


![无缝过渡](http://upload-images.jianshu.io/upload_images/1835430-e7706ffe827b1ea1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


恩，说了这么多，老司机放一下地址：[DWCoreTextLabel](https://github.com/CodeWicky/DWCoreTextLabel)，宝宝们给个star吧~爱你哟~


![爱你哟](http://upload-images.jianshu.io/upload_images/1835430-1a477250d897b4b1.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- - - 
文章到最后都是一样的，喜欢点赞吧~关注吧~乖~话说老司机超爱兔子的~

![乖](http://upload-images.jianshu.io/upload_images/1835430-1fd05923725b25f6.gif?imageMogr2/auto-orient/strip)
