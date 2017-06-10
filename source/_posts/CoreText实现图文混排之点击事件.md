
---
title: CoreText实现图文混排之点击事件
layout: post
date: 2016-05-17 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- CoreText 
- 图文混排
- 点击事件
categories: 图文混排

---

![CoreText实现图文混排之点击事件](http://upload-images.jianshu.io/upload_images/1835430-1e81a37668cb04ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
系列文章：

- [CoreText实现图文混排](http://www.jianshu.com/p/6db3289fb05d)
- [CoreText实现图文混排之点击事件](http://www.jianshu.com/p/51c47329203e)
- [CoreText实现图文混排之文字环绕及点击算法](http://www.jianshu.com/p/e154047b0f98)
- - -
今天呢，我们继续把CoreText图文混排的`点击事件`补充上，这样我们的图文混排也算是圆满了。

<!-- more -->


哦，上一篇的链接在这里
[CoreText实现图文混排](http://www.jianshu.com/p/6db3289fb05d)。所有需要用到的`准备知识`都在上一篇，没有赶上车的朋友可以去补个票~

上正文。
- - - 
## 全部代码
我们知道，CoreText是基于UIView去绘制的，那么既然有UIView，就有

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event方法，我们呢，就是基于这个方法去做点击事件的。

```
通过touchBegan方法拿到当前点击到的点，然后通过坐标判断这个点是否在某段文字上，如果在则触发对应事件。
```
上面呢就是主要思路。接下来呢，我们来详细讲解一下。还是老规矩，先上代码。

```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch * touch = [touches anyObject];
    CGPoint location = [self systemPointFromScreenPoint:[touch locationInView:self]];
    if ([self checkIsClickOnImgWithPoint:location]) {
            return;
    }
    [self ClickOnStrWithPoint:location];
}

-(BOOL)checkIsClickOnImgWithPoint:(CGPoint)location
{
    if ([self isFrame:_imgFrm containsPoint:location]) {
    	NSLog(@"您点击到了图片");
        return YES;
    }
    return NO;
}

-(void)ClickOnStrWithPoint:(CGPoint)location
{
    NSArray * lines = (NSArray *)CTFrameGetLines(self.data.ctFrame);
    CFRange ranges[lines.count];
    CGPoint origins[lines.count];
    CTFrameGetLineOrigins(_frame, CFRangeMake(0, 0), origins);
    for (int i = 0; i < lines.count; i ++) {
        CTLineRef line = (__bridge CTLineRef)lines[i];
        CFRange range = CTLineGetStringRange(line);
        ranges[i] = range;
    }
    for (int i = 0; i < _length; i ++) {
        long maxLoc;
        int lineNum;
        for (int j = 0; j < lines.count; j ++) {
            CFRange range = ranges[j];
            maxLoc = range.location + range.length - 1;
            if (i <= maxLoc) {
                lineNum = j;
                break;
            }
        }
        CTLineRef line = (__bridge CTLineRef)lines[lineNum];        CGPoint origin = origins[lineNum];
        CGRect CTRunFrame = [self frameForCTRunWithIndex:i CTLine:line origin:origin];
        if ([self isFrame:CTRunFrame containsPoint:location]) {  
            NSLog(@"您点击到了第 %d 个字符，位于第 %d 行，然而他没有响应事件。",i,lineNum + 1);//点击到文字，然而没有响应的处理。可以做其他处理
            return;
        }
    }
    NSLog(@"您没有点击到文字");
}

-(BOOL)isIndex:(NSInteger)index inRange:(NSRange)range
{
    if ((index <= range.location + range.length - 1) && (index >= range.location)) {
        return YES;
    }
    return NO;
}

-(CGPoint)systemPointFromScreenPoint:(CGPoint)origin
{
    return CGPointMake(origin.x, self.bounds.size.height - origin.y);
}

-(BOOL)isFrame:(CGRect)frame containsPoint:(CGPoint)point
{
    return CGRectContainsPoint(frame, point);
}

-(CGRect)frameForCTRunWithIndex:(NSInteger)index
                         CTLine:(CTLineRef)line
                         origin:(CGPoint)origin
{
    CGFloat offsetX = CTLineGetOffsetForStringIndex(line, index, NULL);
    CGFloat offsexX2 = CTLineGetOffsetForStringIndex(line, index + 1, NULL);
    offsetX += origin.x;
    offsexX2 += origin.x;
    CGFloat offsetY = origin.y;
    CGFloat lineAscent;
    CGFloat lineDescent;
    NSArray * runs = (__bridge NSArray *)CTLineGetGlyphRuns(line);
    CTRunRef runCurrent;
    for (int k = 0; k < runs.count; k ++) {
        CTRunRef run = (__bridge CTRunRef)runs[k];
        CFRange range = CTRunGetStringRange(run);
        NSRange rangeOC = NSMakeRange(range.location, range.length);
        if ([self isIndex:index inRange:rangeOC]) {
            runCurrent = run;
            break;
        }
    }
    CTRunGetTypographicBounds(runCurrent, CFRangeMake(0, 0), &lineAscent, &lineDescent, NULL);
    CGFloat height = lineAscent + lineDescent;
    return CGRectMake(offsetX, offsetY, offsexX2 - offsetX, height);
}
```
看上去也挺多的，我们还是分段讲解吧。
- - -
## 分段解析
### -touchesBegan
之所以把他放在首位，是因为他作为整个view响应点击事件的`入口`扮演者十分重要的角色。

他负责`接收点击事件`，根据条件将点击事件`分发给不同的对象`去执行相应的响应。

```
///点击方法
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch * touch = [touches anyObject];
    CGPoint location = [self systemPointFromScreenPoint:[touch locationInView:self]];//获取点击位置的系统坐标
    if ([self checkIsClickOnImgWithPoint:location]) {//检查是否点击在图片上，如果在，优先响应图片事件
        return;
    }
    [self ClickOnStrWithPoint:location];//响应字符串事件
}
```
这里老司机还是要解释一下，为什么我要设置成优先响应图片的事件呢？

是这样的，在我们使用的过程中，大部分的场景是如下过程：

- 给整段富文本添加属性，事件等
- 插入图片
- 给图片设置点击事件



正是因为这样，我们可以看出逻辑上图片的响应事件的优先级明显是要高于文字的。即使是一段`文字范围我们赋值了文字的响应事件`，然后在范围中插入了图片并且赋予了图片响应事件，我们往往是`希望图片响应其自己的事件`。同时，不知道你们是否还记得[上一趟车](http://www.jianshu.com/p/6db3289fb05d)我们已经求出了图片的frame，如果优先判断出点击的是图片的话将会`减少很多计算量`，`提高运行效率`。所以我这里将图片的响应优先级定义的高于文字，不过根据需要我们可以定义不同的响应优先级。

搞明白这一点以后，其实逻辑就很简单了。

- 首先呢，先取出当前点击的到屏幕坐标的点。
- 将屏幕坐标转换为系统坐标（不懂得同学快去上一节补课）
- 判断是否点击在图片上
- 如果未点击图片执行点击文字
- - -
### 获取点击坐标
-touchesBegan事件给我们提供了touches这么一个集合。里面装满了UITouch对象。

因为集合是无序的，所以我们通过anyObject取出其中的一个UITouch对象。
UITouch对象的locationInView是专门用来给出UITouch对象在某个View中的坐标的方法，因此我们可以用这个方法来求出当前点击位置的系统坐标。这段比较基础，想画个重点都不知道画哪。
- - -
### 坐标转换
这里用到了第一个工具方法（老司机习惯把写好的方法分类，这些中间方法老司机习惯叫他们工具方法），-(CGPoint)systemPointFromScreenPoint:(CGPoint)origin。

简单的说一句，因为屏幕坐标与系统坐标的不同，我们要将坐标系`统一成系统坐标`，这样才能计算，所以才有了这个坐标转换的方法。其实很简单

```
///坐标转换
/*
 将屏幕坐标转换为系统坐标
 */
-(CGPoint)systemPointFromScreenPoint:(CGPoint)origin
{
    return CGPointMake(origin.x, self.bounds.size.height - origin.y);
}
```
上一讲有坐标系的图，这里我就不细讲了。直接进入下一话题。
- - -
### 点击图片判断
第二个工具方法

-(BOOL)checkIsClickOnImgWithPoint:(CGPoint)location

```
///图片点击检查
/*
 遍历图片frame的数组与点击位置比较，如果在
 范围内则响应的数组中取出对应响应并执行，返
 回yes，否则返回no
 */
-(BOOL)checkIsClickOnImgWithPoint:(CGPoint)location
{
    if ([self isFrame:_imgFrm containsPoint:location]) {
    	NSLog(@"您点击到了图片");
        return YES;
    }
    return NO;
}
```
这里呢，我们用到了第三个工具方法，顺便就说了吧


-(BOOL)isFrame:(CGRect)frame containsPoint:(CGPoint)point

```
///点包含检测
-(BOOL)isFrame:(CGRect)frame containsPoint:(CGPoint)point
{
    return CGRectContainsPoint(frame, point);
}
```
事实上也是调用了系统的一个方法`CGRectContainsPoint()`。这个方法两个参数，一个是frame，一个是point。可以返回point是否在frame中。

不过还是有一点需要注意的。由于`传入的point是系统坐标`（本例中），所以frame我们一定要`传入系统坐标系下的frame`才能正确对应。

这里老司机偷了个懒，直接把上一讲中求得的图片frame改成了一个实例变量，这样在这里的方法中我就能直接调用了。这只是个demo，所以我就怎么方便怎么来了，实际使用中，你可以`把frame保存在数组或字典中`。你问我怎么在数组或字典中保存一个frame这样的结构体？恩，有一个系统类叫`NSValue`，专门针对这种结构体。

如果-(BOOL)isFrame:(CGRect)frame containsPoint:(CGPoint)point返回YES则说明在图片范围内，则`响应图片的点击事件`，

`并且-(BOOL)checkIsClickOnImgWithPoint:(CGPoint)location返回YES`，否则返回NO。

回到上一层，如果-(BOOL)checkIsClickOnImgWithPoint:(CGPoint)location返回YES，则说明`点击的是图片并且已经执行完响应事件`，`直接return结束方法即可`。否则则继续检查是否点击到了文字。
- - -
### 点击文字判断

终于进入重中之重了，点击文字的逻辑了，不过你也别害怕，如果你对上一讲的讲解有了一定的理解的话，这里将变得简单一些。

![逻辑图](http://upload-images.jianshu.io/upload_images/1835430-c25b68eae4815985.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
///字符串点击检查
/*
 实际上接受所有非图片的点击事件，将字符串的每个
 字符取出与点击位置比较，若在范围内则点击到文字
 ，进而检测对应的文字是否响应事件，若存在响应
 */
-(void)ClickOnStrWithPoint:(CGPoint)location
{
    NSArray * lines = (NSArray *)CTFrameGetLines(self.data.ctFrame);//获取所有CTLine
    CFRange ranges[lines.count];//初始化范围数组
    CGPoint origins[lines.count];//初始化原点数组
    CTFrameGetLineOrigins(_frame, CFRangeMake(0, 0), origins);//获取所有CTLine的原点
    for (int i = 0; i < lines.count; i ++) {
        CTLineRef line = (__bridge CTLineRef)lines[i];
        CFRange range = CTLineGetStringRange(line);
        ranges[i] = range;
    }//获取所有CTLine的Range
    for (int i = 0; i < _length; i ++) {//逐字检查
        long maxLoc;
        int lineNum;
        for (int j = 0; j < lines.count; j ++) {//获取对应字符所在CTLine的index
            CFRange range = ranges[j];
            maxLoc = range.location + range.length - 1;
            if (i <= maxLoc) {
                lineNum = j;
                break;
            }
        }
        CTLineRef line = (__bridge CTLineRef)lines[lineNum];//取到字符对应的CTLine
        CGPoint origin = origins[lineNum];
        CGRect CTRunFrame = [self frameForCTRunWithIndex:i CTLine:line origin:origin];//计算对应字符的frame
        if ([self isFrame:CTRunFrame containsPoint:location]) {//如果点击位置在字符范围内，响应时间，跳出循环
            NSLog(@"您点击到了第 %d 个字符，位于第 %d 行，然而他没有响应事件。",i,lineNum + 1);//点击到文字，然而没有响应的处理。可以做其他处理
            return;
        }
    }
    NSLog(@"您没有点击到文字");//没有点击到文字，可以做其他处理
}
```
看上去很多是吧？有没有怕怕的。

仔细看你会发现，有很多代码跟昨天的有相似之处，就是这样，因为这里也`遍历`了每一个CTRun，只不过更加细化到`CTRun中的每个字`。

#### 获取CTLine

```
NSArray * lines = (NSArray *)CTFrameGetLines(self.data.ctFrame);//获取所有CTLine
    CFRange ranges[lines.count];//初始化范围数组
    CGPoint origins[lines.count];//初始化原点数组
    CTFrameGetLineOrigins(_frame, CFRangeMake(0, 0), origins);//获取所有CTLine的原点
```
这四句我就不多说了，获取所有CTLine和其原点。

#### 获取CTLine所表示范围

```
for (int i = 0; i < lines.count; i ++) {
        CTLineRef line = (__bridge CTLineRef)lines[i];
        CFRange range = CTLineGetStringRange(line);
        ranges[i] = range;
    }//获取所有CTLine的Range
```
获取每个CTLine中包含的富文本`在整串富文本中的范围`。将所有CTLine中字符串的范围保存下来放入数组备用。

#### 获取CTRun

```
for (int i = 0; i < _length; i ++)
```
这个for循环用来遍历富文本中的`每一个字符`。下面的代码都是在for循环中的循环体。

```
for (int j = 0; j < lines.count; j ++) {//获取对应字符所在CTLine的index
            CFRange range = ranges[j];
            maxLoc = range.location + range.length - 1;
            if (i <= maxLoc) {
                lineNum = j;
                break;
            }
        }
```
这里又是一层循环，通过当前`字符序号i`与每个`CTLine包含字符的范围`比较来求得当前`计算的是哪个CTLine中的字符`。

```
CTLineRef line = (__bridge CTLineRef)lines[lineNum];//取到字符对应的CTLine
        CGPoint origin = origins[lineNum];
        CGRect CTRunFrame = [self frameForCTRunWithIndex:i CTLine:line origin:origin];//计算对应字符的frame
```

#### 计算CTRun的frame
取得当前字符所在的CTLine并取得该CTLine的原点，同时通过这里的第五个工具方法

-(CGRect)frameForCTRunWithIndex:(NSInteger)index
                         CTLine:(CTLineRef)line
                         origin:(CGPoint)origin
                         
计算当前字符的frame。
分解讲一下这个方法

```
///字符frame计算
/*
 返回索引字符的frame
 
 index：索引
 line：索引字符所在CTLine
 origin：line的起点
*/
-(CGRect)frameForCTRunWithIndex:(NSInteger)index
                         CTLine:(CTLineRef)line
                         origin:(CGPoint)origin
{
    CGFloat offsetX = CTLineGetOffsetForStringIndex(line, index, NULL);//获取字符起点相对于CTLine的原点的偏移量
    CGFloat offsexX2 = CTLineGetOffsetForStringIndex(line, index + 1, NULL);//获取下一个字符的偏移量，两者之间即为字符X范围
    offsetX += origin.x;
    offsexX2 += origin.x;//坐标转换，将点的CTLine坐标转换至系统坐标
    CGFloat offsetY = origin.y;//取到CTLine的起点Y
    CGFloat lineAscent;//初始化上下边距的变量
    CGFloat lineDescent;
    NSArray * runs = (__bridge NSArray *)CTLineGetGlyphRuns(line);//获取所有CTRun
    CTRunRef runCurrent;
    for (int k = 0; k < runs.count; k ++) {//获取当前点击的CTRun
        CTRunRef run = (__bridge CTRunRef)runs[k];
        CFRange range = CTRunGetStringRange(run);
        NSRange rangeOC = NSMakeRange(range.location, range.length);
        if ([self isIndex:index inRange:rangeOC]) {
            runCurrent = run;
            break;
        }
    }
    CTRunGetTypographicBounds(runCurrent, CFRangeMake(0, 0), &lineAscent, &lineDescent, NULL);//计算当前点击的CTRun高度
    offsetY -= lineDescent;
    CGFloat height = lineAscent + lineDescent;
    return CGRectMake(offsetX, offsetY, offsexX2 - offsetX, height);//返回一个字符的Frame
}
```
根据注释就能很轻易的看懂这段代码，不过可能有几个方法不熟悉，我来介绍下。

- CTLineGetOffsetForStringIndex(,,)

获取一行文字中，`指定charIndex字符相对x原点的偏移量`，返回值与第三个参数同为一个值。`如果charIndex超出一行的字符长度则反回最大长度结束位置的偏移量`，如一行文字共有17个字符，哪么返回的是第18个字符的起始偏移，即第17个偏移+第17个字符占有的宽度=第18个起始位置的偏移。因此想求一行字符所占的像素长度时，就可以使用此函数，将charIndex设置为大于字符长度即可。

因为求得的坐标是相对于CTLine原点的偏移量，因此我们要加上CTLine原点的x坐标`获得该点的绝对坐标`。

- CTLineGetGlyphRuns()昨天有介绍过，拿到CTLine中的所有CTRun。

- CTRunGetStringRange()获得CTRun在富文本中的范围
- CTRunGetTypographicBounds(,,,,)获得对应CTRun的尺寸信息

#### 判断点击的文字

中间用了第六个工具方法

-(BOOL)isIndex:(NSInteger)index inRange:(NSRange)range

```
///范围检测
/*
 范围内返回yes，否则返回no
 */
-(BOOL)isIndex:(NSInteger)index inRange:(NSRange)range
{
    if ((index <= range.location + range.length - 1) && (index >= range.location)) {
        return YES;
    }
    return NO;
}
```
这个代码很简单我就不多说了。

通过以上方法，你就拿到了`每一个字符的frame`了。

可以返回至上一层了=。=喘了一口气。。。

接受到字符的frame，还是`判断点击位置是否在frame中`，如果在，则响应点击事件并结束方法。如果没有不在任何一个字符的frame内，则说明没有点击到文字，执行相应的点击事件。

大工告成，到了这里，CoreText做图文混排的点击事件也算是完成了。

最后放一张效果图吧。
![大萌神镇楼](http://upload-images.jianshu.io/upload_images/1835430-6557831b9e067768.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- - -

呐，了却一桩心事。。。

你要是喜欢呢，麻烦你动一动你可爱的小手点击一下喜欢或者关注，毕竟老司机这么爱慕虚荣的人，而且老司机会经常更新的。

哦，这段代码是我自己的解决方案，所以要转载的同学，一定要注明出处哦，这次是一定哦。貌似你不注明我也拦不住你。。。啧啧啧。。。
[http://www.jianshu.com/p/51c47329203e](http://www.jianshu.com/p/51c47329203e)

参考资料：

无

2016年05月16日23点52分

老司机Wicky
