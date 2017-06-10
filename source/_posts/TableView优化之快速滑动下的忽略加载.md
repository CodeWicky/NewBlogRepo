---

title: TableView优化之快速滑动下的忽略加载
layout: post
date: 2017-05-14 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- TableView优化 
- 忽略加载 
categories: 性能优化

---

![TableView优化之快速滑动下的忽略加载](http://upload-images.jianshu.io/upload_images/1835430-13da7ca8ce7602d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


系列文章：

- [TableView优化之高度缓存功能](http://www.jianshu.com/p/2b192257276f)

- [TableView优化之加载图片的优化逻辑](http://www.jianshu.com/p/328e503900d0)

- [TableView优化之快速滑动下的忽略加载](http://www.jianshu.com/p/0b020518de5e)


- - -

最近在搞什么，所以就顺手写点什么咯~

这两天一直在搞一个TableView的工具类，因为觉得这个东西写完可以一劳永逸，所以就去搞了一下，主要是有助于TableView的快捷开发。没什么好废话的了，直接说事吧=。=

在今天的博客中你可能会看到：
- VVeboTableView中Cell加载逻辑的解析
- TableView代码解耦的基本思路

恩，东西不多，一点一点说~

<!-- more -->

- - -  
### VVeboTableView
其实这是VVebo项目中作者分享剥离的一个Demo，来告诉我们他是怎么优化TableView的流畅性的。

那么VVebo是什么呢？看名字你就猜吧，像不像微博，是的，它就是一款新浪微博的第三方客户端，当年还是有很多人追捧的，不过后来新浪逐渐收回开发接口导致很多功能无法实现就把VVebo给坑了。

那么为什么VVebo使用率那么高呢？一方面是当时新浪微博客户端的确不行，另一方面VVebo简约的风格和流畅的体验俘获了一大批用户。所以今天我们就来探究一下他是如何做到TableView的丝滑体验的。

首先你可以在[这里现在一份源码](https://github.com/johnil/VVeboTableViewDemo)，毕竟源码面前没有秘密。

在老司机看来，作者最有效的优化分为4部分：

- TableViewCell圆角优化
- 缓存行高
- 相对固定的图片及文字采用CoreText绘制
- TableView加载数据逻辑优化
- - -

#### 圆角
这部分作者的优化很简单，**他没有画圆角！**圆角是TableViewCell的帧率杀手大家都知道吧，所以人家根本就没有画圆角。他是怎么做的呢？`覆盖了与背景色同色的圆角图片`，简单粗暴，果然是个心机boy。

不过关于圆角的优化，还是有更好的解决办法的，[在这里](http://www.jianshu.com/p/f970872fdc22)。不想看的话我给你总结一下，就两点：
- 别冤枉cornerRadius，问题不在它。而**在于maskToBounds**。普通的UIView绘制圆角时并不需要maskToBounds属性。也就是普通的视图圆角对卡顿没有影响。
- 既然有普通就有特殊：UIImageView和UILabel以及我还没有发现的=。=对于UIImage的处理建议先借助CoreGraphic处理图片吧，**直接绘制一个带圆角的图片给ImageView吧**。对于Label没有太好的优化方案，是在不行只能CoreText了。其实你会发现，UILable这个控件对中文**十！分！不！友！好！**很多细节上中文跟英文或者字符会有很大的差异，但是你有不能不用他，好气哦=。=

- - -
#### 缓存行高
这部分内容老司机在上一期讲述过不定高cell行高缓存的必要性及缓存的方法，这里不再赘述。
- - -
#### CoreText绘制文本
首先，复杂的`层级关系同样会给cell在绘制时添加很大的负担`，这点是毋庸置疑的，所以VVebo的作者选择了将一些相对重复性很大的视图选择使用CoreText和CoreGraphic技术直接绘制在一个视图上，这样就`减少了视图的层级`，为流畅性又添了一份可能。CoreText绘制文本的和图片的技术你可以在老司机的[CoreText实现图文混排系列](http://www.jianshu.com/p/6db3289fb05d)中得到详细的实现方法，想看的去看吧。
- - -
#### TableView加载数据逻辑优化
到现在为止终于要讲点之前没有说过的了=。=

说以下主体思路，VVebo的作者认为，**当用户快速滑动的时候，事实上他对滑动过程中的内容是不关心的**，他只关心滚动结束处的内容，那么用户不关心的内容她就选择了不加载。

这是他的主体思路，来看下这部分的实现代码：

```
- (void)drawCell:(VVeboTableViewCell *)cell withIndexPath:(NSIndexPath *)indexPath{
    NSDictionary *data = [datas objectAtIndex:indexPath.row];
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    //清除cell内容，解决复用问题
    [cell clear];
    cell.data = data;
  //判断如果needLoadArr中含有需要加载的indexPath而当前indexPath又不在其中的时候，则不绘制cell直接返回
    if (needLoadArr.count>0&&[needLoadArr indexOfObject:indexPath]==NSNotFound) {
        [cell clear];
        return;
    }
  //判断如果scrollToToping为真的时候（及点击状态栏快速回到TableView顶部的时候）不绘制cell
    if (scrollToToping) {
        return;
    }
    //上面都没问题的话，绘制cell
    [cell draw];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    VVeboTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"cell"];
    if (cell==nil) {
        cell = [[VVeboTableViewCell alloc] initWithStyle:UITableViewCellStyleDefault
                                         reuseIdentifier:@"cell"];
    }
    [self drawCell:cell withIndexPath:indexPath];
    return cell;
}

- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView{
    [needLoadArr removeAllObjects];
}

//按需加载 - 如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定3行加载。
- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset{
//取出滚动停止时展示的第一个cell的indexPath
    NSIndexPath *ip = [self indexPathForRowAtPoint:CGPointMake(0, targetContentOffset->y)];
//取出当前展示的第一个cell的indexPath
    NSIndexPath *cip = [[self indexPathsForVisibleRows] firstObject];
    NSInteger skipCount = 8;
//如果两者之间差距很大则认为滑动速度很快，中间用户都不关心，直接把滚动停止时的展示的cell加入到needLoadArr数组中
    if (labs(cip.row-ip.row)>skipCount) {
        NSArray *temp = [self indexPathsForRowsInRect:CGRectMake(0, targetContentOffset->y, self.width, self.height)];
        NSMutableArray *arr = [NSMutableArray arrayWithArray:temp];
//根据滚动方向在前或后额外添加三个需要展示的cell，这样看起来好像更加平滑的样子
        if (velocity.y<0) {
            NSIndexPath *indexPath = [temp lastObject];
            if (indexPath.row+3<datas.count) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+1 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+3 inSection:0]];
            }
        } else {
            NSIndexPath *indexPath = [temp firstObject];
            if (indexPath.row>3) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-3 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-1 inSection:0]];
            }
        }
        [needLoadArr addObjectsFromArray:arr];
    }
}

- (BOOL)scrollViewShouldScrollToTop:(UIScrollView *)scrollView{
    scrollToToping = YES;
    return YES;
}

- (void)scrollViewDidEndScrollingAnimation:(UIScrollView *)scrollView{
    scrollToToping = NO;
    [self loadContent];
}

- (void)scrollViewDidScrollToTop:(UIScrollView *)scrollView{
    scrollToToping = NO;
    [self loadContent];
}

//用户触摸时第一时间加载内容
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    if (!scrollToToping) {
        [needLoadArr removeAllObjects];
        [self loadContent];
    }
    return [super hitTest:point withEvent:event];
}

- (void)loadContent{
    if (scrollToToping) {
        return;
    }
    if (self.indexPathsForVisibleRows.count<=0) {
        return;
    }
    if (self.visibleCells&&self.visibleCells.count>0) {
        for (id temp in [self.visibleCells copy]) {
            VVeboTableViewCell *cell = (VVeboTableViewCell *)temp;
            [cell draw];
        }
    }
}
```

其实是就100行代码，思路还是很清晰明了的。作者主要是通过
`-drawCell:withIndexPath:`这个方法来控制cell的绘制行为的。我们看看他做了什么？

首先他**cell调用了clear方法**，这是VVeboTableViewCell中作者自己实现的方法，用于清除cell上面展示的内容，这样可以**避免因cell重用而导致没有绘制**的cell会显示之前的内容的问题。然后是判断**needLoadArr**中是否包含有当前indexPath，若没有返回。继续判断当前TableView是否处于**快速回到顶部的过程中**，如果是的话也不绘制。**最后上述条件都满足的时候再进行cell的绘制**。

所以重点来了，needLoadArr什么时候添加的元素？如何获取到TableView快速回到顶部的时间点？

>第二点好说，点击状态栏的时候，TableView会询问代理  `- scrollViewShouldScrollToTop:         `只有返回YES的时候才会快速回到顶部，这时我们可以在这捕获到这个状态。但是可以看到作者并没有在这选择添加顶部可能要展示的cell进needLoadArr数组，那么当他滚动到顶部的时候我们要将顶部的cell进行直接更新，所以通过`- scrollViewDidEndScrollingAnimation:`和`- scrollViewShouldScrollToTop:`两个代理拿到到达顶部的状态后直接更新当前cell。

回过头来我们说下第一点，needLoadArr是怎么操作呢？
我们知道我们是要判断TableView快速滑动，那我们怎么拿到这个行为呢？要知道没有什么代理是直接反应滚动速度的，这里作者很取巧的用到了`-scrollViewWillEndDragging:withVelocity:targetContentOffset:`这个代理。
这个代理在手指即将结束拖动的时候出发，他会告诉外界当前的速度及这次会滚动到的位置。

>所以作者在这里判断了目标位置与当前位置相差间隔，如果很大的话则认为中间内容不需加载，直接添加目标位置的内容进入数组。

恩，以上就是VVebo作者对数据加载逻辑的优化。

这是依靠着上述四点，VVebo才获得了完美的滑动体验，其`思路也是我们开发中可以学习和借鉴的`。

- - -
### TableView解耦
这部分内容也不是什么新鲜事，也是比较靠谱的一个思路。当然了这部分内容不是对性能的优化，而是对代码的优化。

天天写TableView里面的代理是不是很烦人啊，**千篇一律又不能不写**。所以想一个方法只写一次以后拿来直接用吧=。=


![效果图](http://upload-images.jianshu.io/upload_images/1835430-3b67fb2cbb3cdccc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![真机不卡！真机不卡！真机不卡！重要的事情说三遍](http://upload-images.jianshu.io/upload_images/1835430-88033ea7bfde7ee3.gif?imageMogr2/auto-orient/strip)


放一个效果图，老司机写的控制器里面看不到任何一个TableView代理然而还是能正常显示并实现很多功能。

但是代码怎么可能不写，只是我在别的地方写过了，并且花了大把时间进行解耦，让每一个TableView都能拿来就直接使用。

那么这个解耦的类我们要怎么写呢？

好的，我们来新建一个文件。
![helper类](http://upload-images.jianshu.io/upload_images/1835430-690063c9d0cecc7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个类只需要一个属性，是一个数组。就是你平常写TableView的时候的数据源。

然后在.m中我们就可以像平常写TableView一样在这里面写代理了。

![假装写了两个代理](http://upload-images.jianshu.io/upload_images/1835430-eb7262ea8c5d87f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

无视我的cell和model，嫌累没创建=。=

最后在VC中把TableView的dataSource设成Helper就好了。


![无视我这代码，我就是给你展现个逻辑，细写嫌累](http://upload-images.jianshu.io/upload_images/1835430-8d9a3f231a8ac39d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**重点是别忘了持有helper类**。tableView对dataSource是弱引用，如果不持有helper就被释放了。
就是这么一个思路。的确该写你都写了，不过好处就是你以后把helper类拿到另一个工程还可以直接用。

恩，思路就是这么简单的一个思路，不过你可以把你的helper类写的功能更加丰富一些。比如说我的helper类。老司机添加了**高度缓存、滚动优化**等优化功能，并且对**选择、展示动画、无数据占位图**等常用功能都进行了支持。而且老司机也在不断的丰富helper类的功能。

只放一个版本更新记录吧，代码放不下=。=
>/**
 DWTableViewHelper
 TableView工具类
 抽出TableView代理，减小VC压力，添加常用代理映射
 
> version 1.0.0
 添加常用代理映射
 添加helper基础属性
 
> version 1.0.1
 去除注册，改为更适用的重用模式
 
> version 1.0.2
 添加多分组模式
 
> version 1.0.3
 添加选择模式及相关api
 
> version 1.0.4
 添加helper设置cell类型及复用标识
 
> version 1.0.5
 将cell的基础属性提出协议，helper与model同时遵守协议
 
> version 1.0.6
 修正占位视图展示时机，提供两个刷新列表扩展方法，提供展示、隐藏占位图接口
 
> version 1.0.7
 添加选则模式下单选多选控制
 
> version 1.0.8
 补充组头视图、尾视图行高代理映射并简化代理链
 
> version 1.0.9
 cell基类添加父类实现强制调用宏、断言中给出未能加载的cell类名
 
> version 1.1.0
 改变cell划线机制，改为系统分割线，添加分割线归0方法
 添加自动行高计算并缓存
 cell添加xib支持
 修复选择模式选中后关闭再次开启选择同一个无法选中bug
 更换去除选择背景方式，解决与选择模式的冲突
 映射所有代理
 
> version 1.1.1
 添加自适应模式最小行高限制及最大行高设置
 添加数据源的容错机制，但这并不是你故意写错的理由=。=
 添加屏幕判断，当位置方向时，默认返回竖屏
 额外补充动画代理、支持CAAnimation及DWAnimation
 
> version 1.1.2
 展示动画逻辑修改，DWAnimation动画展示方法替换
 
> version 1.1.3
 滚动优化模式添加
 高速忽略模式完成
 懒加载模式完成
 懒加载模式动画隐藏，更加平滑，修复刷新bug。
 有没有美工妹子给切几张占位图。。我做的图太丑了。。
 
> */

是的，所以说你玩去那可以**写一个什么都能做的Helper**。

正如我最开始的效果图。如果你想看看我还对Helper做了什么你可以去我的仓库上面看[DWTableViewHelper](https://github.com/CodeWicky/DWTableViewHelper)。

你想直接用也可以，你可以去GitHub上面直接托一份，也可以用cocoaPods集成：
```
      pod 'DWTableViewHelper', '~> 1.1.2'
```
DWTableViewHelper类当前为1.1.2版本，滚动优化在1.1.3版本pod还没有发，因为在测试看有没有什么bug，而且老司机做的图有的丑，急需会美工的妹子帮我切两张图，汉子也行，`愿意帮忙的私信我`=。=

如果你想看看老司机的所有pods项目的话，你也可以打开终端，输入
```
pod search wicky
```


![pod search wicky](http://upload-images.jianshu.io/upload_images/1835430-8a2f5542d40a2965.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，双击666，加波关注，点波star，老铁没毛病！
![老铁没毛病](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1494754370835&di=3edf2dea5efdf9b4da0427f1d1a69b16&imgtype=0&src=http%3A%2F%2Fnews.k618.cn%2Fsociety%2F201703%2FW020170325684515461265.png)
