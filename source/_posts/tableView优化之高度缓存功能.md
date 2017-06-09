---

title: TableView优化之高度缓存
layout: post
date: 2016-06-04 07:33:44
tags: 
- TableView 
- 高度缓存
categories: 性能优化

---

![TableView优化之高度缓存](http://upload-images.jianshu.io/upload_images/1835430-a69cecb9aaa3f2d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
系列文章：

- [TableView优化之高度缓存功能](http://www.jianshu.com/p/2b192257276f)

- [TableView优化之加载图片的优化逻辑](http://www.jianshu.com/p/328e503900d0)
- - -

日常开发中，tableView的使用率很高，所以相对tableView的优化来说可以做很多很多的事情。很多情况下，我们的每一个cell都没有一个固定的高度，而是根据cell里面的内容自适应高度的。那么每次当我们cell将要出现在屏幕的时候，系统都会去计算cell的高度。如果说我能通过某种手段，在首次计算的时候，将每个cell对应的高度保存下载，当下次需要用到cell高度的时候再从保存的地方取出，从而减少了计算量，来达到优化的目的。

因此应运而生了这套高度缓存的算法。

在此声明，这套算法`不是老司机原创`，是整合并优化了外国一位大神的源码。

**再次声明，**
这原来是国人算法，扬我国威啊，原型名
UITableView-FDTemplateLayoutCell。

<!-- more -->

- - -
这篇博客中你可能会用到

- category 、subclass、protocol之间的区别
- 自动布局相关
- runTime动态绑定
- tableViewCell重用的相关

恩，其实东西并不多，重要的还是一个思想。


---

老规矩，还是先放全部代码。

```
#import "UITableView+HeightCache.h"
#import <objc/runtime.h>
@implementation UITableView (HeightCache)

#pragma mark ---接口方法---
-(CGFloat)DW_CalculateCellWithIdentifier:(NSString *)identifier
                               indexPath:(NSIndexPath *)indexPath
                           configuration:(void(^)(id cell))configuration
{
    if(self.bounds.size.width != 0)//防止初始宽度为0（如autoLayout初次加载时）
    {
        if (!identifier.length || !indexPath) {//非空判断
            return 0;
        }
        NSString * key = [self.cache makeKeyWithIdentifier:identifier indexPath:indexPath];//制作key
        if ([self.cache existInCacheByKey:key]) {//如果key存在
            return [self.cache heightFromCacheWithKey:key];//从字典中取出高
        }
        CGFloat height = [self DW_CalCulateCellWithIdentifier:identifier configuration:configuration];//不存在则计算高度
        [self.cache cacheHeight:height byKey:key];//并缓存
        return height;
    }
    return 0;
}
-(void)DW_RemoveHeightCacheWithIdentifier:(NSString *)identifier
                                indexPath:(NSIndexPath *)indexPath
                             numberOfRows:(NSInteger)rows
{
    [self.cache removeHeightByIdentifier:identifier indexPath:indexPath numberOfRows:rows];
}
-(void)DW_RemoveAllHeightCache
{
    [self.cache removeAllHeight];
}
-(void)DW_InsertCellToIndexPath:(NSIndexPath *)indexPath
                 withIdentifier:(NSString *)identifier
                   numberOfRows:(NSInteger)rows
{
    [self.cache insertCellToIndexPath:indexPath withIdentifier:identifier numberOfRows:rows toDictionaryForCache:self.cache.dicHeightCurrent];
}
-(void)DW_MoveCellFromIndexPath:(NSIndexPath *)sourceIndexPath
    sourceIndexPathNumberOfRows:(NSInteger)sourceRows
                    toIndexPath:(NSIndexPath *)destinationIndexPath
destinationIndexPathNumberOfRows:(NSInteger)destinationRows
                 withIdentifier:(NSString *)identifier
{
    [self.cache moveCellFromIndexPath:sourceIndexPath sourceSectionNumberOfRows:sourceRows toIndexPath:destinationIndexPath destinationSectionNumberOfRows:destinationRows withIdentifier:identifier];
}
#pragma mark ---工具方法---
///从重用池中返回计算用的cell
-(__kindof UITableViewCell  *)DW_CalculateCellWithIdentifier:(NSString *)identifier
{
    if (!identifier.length) {
        return nil;
    }
    NSMutableDictionary <NSString * ,UITableViewCell *> *DicForTheUniqueCalCell = objc_getAssociatedObject(self, _cmd);//利用runtime取出tableV绑定的存有cell的字典
    if (!DicForTheUniqueCalCell) {
        DicForTheUniqueCalCell = [NSMutableDictionary dictionary];//如果取不到则新建并绑定
        objc_setAssociatedObject(self, _cmd, DicForTheUniqueCalCell, OBJC_ASSOCIATION_RETAIN_NONATOMIC);//动态绑定（绑定目标，关键字，绑定者，策略）
    }
    //以上只是为了只绑定一个字典，类比懒加载
    UITableViewCell * cell = DicForTheUniqueCalCell[identifier];
    if (!cell) {
        cell = [self dequeueReusableCellWithIdentifier:identifier];//从重用池中取一个cell用来计算，必须以本方式从重用池中取，若以indexPath方式取由于-heightForRowAtIndexPath方法会造成循环。
        cell.contentView.translatesAutoresizingMaskIntoConstraints = NO;//开启约束
        cell.JustForCal = YES;//标记只用来计算
        DicForTheUniqueCalCell[identifier] = cell;
    }
    //同上，保证只有一个用来计算的cell
    return cell;
}

///根据重用表示取出cell并操作cell后，计算高度
-(CGFloat)DW_CalCulateCellWithIdentifier:(NSString *)identifier
                           configuration:(void(^)(id cell))configuration
{
    if (!identifier.length) {
        return 0;
    }
    UITableViewCell * cell = [self DW_CalculateCellWithIdentifier:identifier];
    [cell prepareForReuse];//放回重用池
    if (configuration) {
        configuration(cell);//对cell进行操作
    }    
    return [self DW_CalculateCellHeightWithCell:cell];
}

///根据cell计算cell的高度
-(CGFloat)DW_CalculateCellHeightWithCell:(UITableViewCell *)cell
{
    CGFloat width = self.bounds.size.width;
    //根据辅助视图校正width
    if (cell.accessoryView) {
        width -= cell.accessoryView.bounds.size.width + 16;
    }
    else
    {
        static const CGFloat accessoryWidth[] = {
            [UITableViewCellAccessoryNone] = 0,
            [UITableViewCellAccessoryDisclosureIndicator] = 34,
            [UITableViewCellAccessoryDetailDisclosureButton] = 68,
            [UITableViewCellAccessoryCheckmark] = 40,
            [UITableViewCellAccessoryDetailButton] = 48
        };
        width -= accessoryWidth[cell.accessoryType];
    }
    CGFloat height = 0;
    if (!cell.NoAutoSizing && width > 0) {//如果不是非自适应模式则添加约束后计算约束后高度
        NSLayoutConstraint * widthConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1.0 constant:width];//创建约束
        [cell.contentView addConstraint:widthConstraint];//添加约束
        height = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;//计算高度
        [cell.contentView removeConstraint:widthConstraint];//移除约束
    }
    if (height == 0) {//如果约束错误可能导致计算结果为零，则以自适应模式再次计算
        height = [cell sizeThatFits:CGSizeMake(width, 0)].height;
    }
    if (height == 0) {//如果计算仍然为0，则给出默认高度
        height = 44;
    }
    if (self.separatorStyle != UITableViewCellSeparatorStyleNone) {//如果不为无分割线模式则添加分割线高度
        height += 1.0 /[UIScreen mainScreen].scale;
    }
    return height;
}

#pragma mark ---setter、getter---
-(HeightCache *)cache//懒加载形式
{
    HeightCache * cacheTemp = objc_getAssociatedObject(self, _cmd);
    if (!cacheTemp) {
        cacheTemp = [HeightCache new];
        objc_setAssociatedObject(self, _cmd, cacheTemp, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return cacheTemp;
}
-(void)setCache:(HeightCache *)cache
{
    objc_setAssociatedObject(self, @selector(cache), cache, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
@end


@implementation UITableViewCell (HeightCacheCell)
#pragma mark ---setter、getter---
-(BOOL)NoAutoSizing
{
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}
-(void)setNoAutoSizing:(BOOL)NoAutoSizing
{
    objc_setAssociatedObject(self, @selector(NoAutoSizing), @(NoAutoSizing), OBJC_ASSOCIATION_RETAIN);//关键字用getter的方法名，为保持关键字一致
}
-(BOOL)JustForCal
{
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}
-(void)setJustForCal:(BOOL)JustForCal
{
    objc_setAssociatedObject(self, @selector(JustForCal), @(JustForCal), OBJC_ASSOCIATION_RETAIN);
}
@end

@implementation HeightCache
///制作key
-(NSString *)makeKeyWithIdentifier:(NSString *)identifier
                         indexPath:(NSIndexPath *)indexPath
{
    return [NSString stringWithFormat:@"%@S%ldR%ld",identifier,indexPath.section,indexPath.row];
}

///高度是否存在
-(BOOL)existInCacheByKey:(NSString *)key
{
    NSNumber * value = [self.dicHeightCurrent valueForKey:key];
    return (value && ![value isEqualToNumber:@-1]);
}

///取出缓存的高度
-(CGFloat)heightFromCacheWithKey:(NSString *)key
{
    NSNumber * value = [self.dicHeightCurrent valueForKey:key];
    if ([self is64bit]) {
        return [value doubleValue];
    }
    return [value floatValue];
}

///64位判断
- (BOOL)is64bit
{
#if defined(__LP64__) && __LP64__
    return YES;
#else
    return NO;
#endif
}

///高度缓存
-(void)cacheHeight:(CGFloat)height
             byKey:(NSString *)key
{
    [self.dicHeightCurrent setValue:@(height) forKey:key];
}

///根据key删除缓存
-(void)removeHeightByIdentifier:(NSString *)identifier
                      indexPath:(NSIndexPath *)indexPath
                   numberOfRows:(NSInteger)rows
{
    if (indexPath.row < rows) {
        for (int i = 0; i < rows - 1 - indexPath.row; i ++) {
            NSIndexPath * indexPathA = [NSIndexPath indexPathForRow:indexPath.row + i inSection:indexPath.section];
            NSLog(@"%ld,%ld",indexPathA.row,indexPathA.section);
            NSIndexPath * indexPathB = [NSIndexPath indexPathForRow:indexPath.row + i + 1 inSection:indexPath.section];
            NSLog(@"%ld,%ld",indexPathB.row,indexPathB.section);
            [self exchangeValueForIndexPathA:indexPathA andIndexPathB:indexPathB withIdentifier:identifier dictionary:self.dicHeightCacheH];
            [self exchangeValueForIndexPathA:indexPathA andIndexPathB:indexPathB withIdentifier:identifier dictionary:self.dicHeightCacheV];
        }
        NSIndexPath * indexPathC = [NSIndexPath indexPathForRow:rows - 1 inSection:indexPath.section];
        NSString * key = [self makeKeyWithIdentifier:identifier indexPath:indexPathC];
        [self.dicHeightCacheH removeObjectForKey:key];
        [self.dicHeightCacheV removeObjectForKey:key];
    }
}

///删除所有缓存
-(void)removeAllHeight
{
    [self.dicHeightCacheH removeAllObjects];
    [self.dicHeightCacheV removeAllObjects];
}

///插入cell是插入value
-(void)insertCellToIndexPath:(NSIndexPath *)indexPath
            withNumberOfRows:(NSInteger)rows
                heightNumber:(NSNumber *)height
                  identifier:(NSString *)identifier
        toDictionaryForCache:(NSMutableDictionary *)dic
{
    if (indexPath.row < rows + 1) {
        [self insertCellToIndexPath:indexPath withIdentifier:identifier numberOfRows:rows toDictionaryForCache:dic];
        NSString * key = [self makeKeyWithIdentifier:identifier indexPath:indexPath];
        [dic setValue:height forKey:key];
    }
}
-(void)insertCellToIndexPath:(NSIndexPath *)indexPath withIdentifier:(NSString *)identifier numberOfRows:(NSInteger)rows toDictionaryForCache:(NSMutableDictionary *)dic
{
    if (indexPath.row < rows + 1) {
        for (int i = 0; i < rows - indexPath.row; i ++) {
            NSIndexPath * indexPathA = [NSIndexPath indexPathForRow:rows - i inSection:indexPath.section];
            NSIndexPath * indexPathB = [NSIndexPath indexPathForRow:rows - i - 1 inSection:indexPath.section];
            [self exchangeValueForIndexPathA:indexPathA andIndexPathB:indexPathB withIdentifier:identifier dictionary:dic];
        }
    }
}
///移动cell时交换value
-(void)moveCellFromIndexPath:(NSIndexPath *)sourceIndexPath
   sourceSectionNumberOfRows:(NSInteger)sourceRows
                 toIndexPath:(NSIndexPath *)destinationIndexPath
destinationSectionNumberOfRows:(NSInteger)destinationRows
              withIdentifier:(NSString *)identifier
{
    if (sourceIndexPath.section == destinationIndexPath.section) {
        [self moveCellInSectionFromIndexPath:sourceIndexPath toIndexPath:destinationIndexPath withIdentifier:identifier];
    }
    else
    {
        [self moveCellOutSectionFromIndexPath:sourceIndexPath sourceSectionNumberOfRows:sourceRows toIndexPath:destinationIndexPath destinationSectionNumberOfRows:destinationRows withIdentifier:identifier];
    }
}
///组内移动
-(void)moveCellInSectionFromIndexPath:(NSIndexPath *)sourceIndexPath
                          toIndexPath:(NSIndexPath *)destinationIndexPath
                       withIdentifier:(NSString *)identifier
{
    NSInteger rowA = sourceIndexPath.row;
    NSInteger rowB = destinationIndexPath.row;
    for (int i = 0; i < (MAX(rowA, rowB) - MIN(rowA, rowB)); i ++) {
        NSIndexPath * indexPathA = [NSIndexPath indexPathForRow:MIN(rowA, rowB) + i inSection:sourceIndexPath.section];
        NSIndexPath * indexPathB = [NSIndexPath indexPathForRow:MIN(rowA, rowB) + i + 1 inSection:sourceIndexPath.section];
        [self exchangeValueForIndexPathA:indexPathA andIndexPathB:indexPathB withIdentifier:identifier dictionary:self.dicHeightCacheV];
        [self exchangeValueForIndexPathA:indexPathA andIndexPathB:indexPathB withIdentifier:identifier dictionary:self.dicHeightCacheH];
    }
}
///组外移动
-(void)moveCellOutSectionFromIndexPath:(NSIndexPath *)sourceIndexPath
             sourceSectionNumberOfRows:(NSInteger)sourceRows
                           toIndexPath:(NSIndexPath *)destinationIndexPath
        destinationSectionNumberOfRows:(NSInteger)destinationRows
                        withIdentifier:(NSString *)identifier
{
    NSNumber * numberH;
    NSNumber * numberV;
    NSLog(@"%ld",sourceIndexPath.row);
    if (sourceIndexPath.row < sourceRows) {
        NSString * key = [self makeKeyWithIdentifier:identifier indexPath:sourceIndexPath];
        numberH = self.dicHeightCacheH[key];
        numberV = self.dicHeightCacheV[key];
        [self removeHeightByIdentifier:identifier indexPath:sourceIndexPath numberOfRows:sourceRows];
    }
    NSLog(@"%ld,%ld",destinationIndexPath.row,destinationIndexPath.section);
    [self insertCellToIndexPath:destinationIndexPath withNumberOfRows:destinationRows heightNumber:numberH identifier:identifier toDictionaryForCache:self.dicHeightCacheH];
    [self insertCellToIndexPath:destinationIndexPath withNumberOfRows:destinationRows heightNumber:numberV identifier:identifier toDictionaryForCache:self.dicHeightCacheV];
}
///根据indexPath交换两个Key
-(void)exchangeValueForIndexPathA:(NSIndexPath *)indexPathA
                    andIndexPathB:(NSIndexPath *)indexPathB
                   withIdentifier:(NSString *)identifier
                       dictionary:(NSMutableDictionary *)dic
{
    NSString * keyA = [self makeKeyWithIdentifier:identifier indexPath:indexPathA];
    NSString * keyB = [self makeKeyWithIdentifier:identifier indexPath:indexPathB];
    NSNumber * Temp = dic[keyA];
    dic[keyA] = dic[keyB];
    dic[keyB] = Temp;
}
#pragma mark ---懒加载---
-(NSMutableDictionary *)dicHeightCacheH
{
    if (!_dicHeightCacheH) {
        _dicHeightCacheH = [NSMutableDictionary dictionary];
    }
    return _dicHeightCacheH;
}
-(NSMutableDictionary *)dicHeightCacheV
{
    if (!_dicHeightCacheV) {
        _dicHeightCacheV = [NSMutableDictionary dictionary];
    }
    return _dicHeightCacheV;
}
-(NSMutableDictionary *)dicHeightCurrent//根据系统状态返回对应字典
{
    return UIDeviceOrientationIsPortrait([UIDevice currentDevice].orientation)?self.dicHeightCacheV:self.dicHeightCacheH;
}
@end

```

``这么多，你这个骗子！``
你是不是这么想得。

别慌，东西不多，因为这是一个category，复用性非常高，所以老司机想尽量减少文件个数，这样集成的时候也方便不是。

所以，老司机`把三个类写在一个文件里了`。

之前有人问过我把几个类写在一个文件中`有什么好处么`？

老司机目前只能说，`除了看上去装逼点，别的没什么卵用`。

可能是老司机的理解不深，不过为了集成方便我还是写在一个文件里了。

废话不多说，分段讲解吧。
- - -
## 分段解析
### 为什么选择category而不是subclass

我想很多人都会有疑问，为什么选择category而不是继承，毕竟category不能添加属性,用起来不是很方便。说到这又要老生常谈了。

>Protocol只是声明一套接口，并不能提供具体实现，变相的也算是一种抽象基类的实现方式（OC本身语法并不支持抽象基类）。

>Category可以为已有的类提供额外的接口和具体的实现。

>Protocol只能提供一套公用的接口声明，并不能提供具体实现，它的行为是，我只负责声明，而不管谁去实现，去如何实现。这样的话，我定义一套接口，可以使任意的类都用不同的方式去实现接口中的方法，就是为遵守了protocol的类提供了一些额外访问这个类的一些接口，像delegate和dataSource用protocol实现是最好的。

>Category是对一个功能完备的类的一种补充、扩展，就像一个东西基本功能都完成了，可以用category为这个类添加不同的组件，使得这个类能够适应不同情况的需求（但是这些不同需求最核心的需求要一致）。当然，当某个类非常大的时候，使用category可以按照不同的功能将类的实现分在不同的模块中。还有，虽然category可以访问已有类的实例变量，但不能创建新的实例变量，如果要创建新的实例变量，请使用继承。

>继承，它基于Protocol和Category之间，既可以像protocol一样只提供纯粹的接口，也可以像Category一样提供接口的完整实现，可以自由定义类的实例变量（这一点，Protocol倒是可以声明实例变量，但是也仅仅是声明而已），而且继承还可以对类以后的方法进行改写，所以继承的力量是最强大的。

>在iOS开发中，继承是完全可以完成protocol和category的功能的，那么在开发过程中多多使用继承体系可好？

>需要注意的是使用继承还有很大的代价问题。使用继承来进行扩展是一种耦合度很高的行为，对父类可以说是完全依赖，如果继承体系太过复杂，会造成难以维护的问题。如果仅仅只是对类进行扩展，并不建议使用继承，毕竟使用protocol和category是很简单、轻松的。除此之外，在开发过程中，我们应该尽量将界面、功能相似的类的代码提取到基类里面，然后各个子类继承自这个基类，实现各自的其他特殊部分。这样可以大大的优化代码，如果需要修改的话，只需要这倒对应子类修改即可。

是不是感觉老司机屌屌的，啧啧啧，我百度的。

我选择category就一个原因，扩展方便，二次开发也方便。

---

### 类目拆解
老司机说过，这一坨代码是三个类写在了一个文件里，他们都是什么呢？

- UITableView的category
- UITableViewCell的category
- 一个NSObject的子类

他们分别是做什么的呢？

首先，给UITableView添加category是因为为了`实现高度缓存`，我的方案是在计算高度的时候就模拟数据填充，从而计算出该cell的高度，所以，tableView应该有填充数据和计算高度的方法。故为其添加分类。

而那个继承于NSObject的子类就是用来`存储计算出来的高度的`，这样当下次需要计算的时候直接从这里取出即可。

至于那个UITableViewCell的类目是为了`给cell添加两个标识符`，一个用来判断当前cell是否需要一autolayout进行绘制，另一个是用来区分计算用的cell还是展示用的cell。这点现在可能你还不懂，一会我们会做相应介绍。

---
### UITableViewCell的category

为什么先说这个类目呢，因为这个类目的内容最少，而且只用到了runtime的动态绑定，可以拿出来单独介绍一下runtime的相关知识。

#### 什么是runtime
run，运行，time，时间，那么runtime就是运行时。恩，简单不？
然后我们说说。。。诶诶诶，别打别打，开个玩笑。
runtime，简称运行时，是`系统在运行期间的一些机制`。而对于OC来讲呢，其中最重要的就是`消息机制`。

C语言呢，我们`调用函数`，在`编译期`就已经确定了要调用那个函数，而且整个过程是`顺序执行`的。

而在OC中呢，我们是讲`消息发送`的。而且我们是等待某个信号`触发才执行`代码的。我们知道OC事实上是基于C的，那他是怎么实现这套转换的呢？就是`通过runtime去实现`的。

不信？不信跟我来做个试验。

新开一个工程，删掉所有文件，只`留下info.plist和main.m`。并且将引入的头文件删除掉。

再`建一个类`，里面随便写一个方法的声明和实现。

然后`在main.m中引入这个类`，`初始化并调用刚才声明的方法`，如下图。


![新建一个工程](http://upload-images.jianshu.io/upload_images/1835430-7d31aa53377e7333.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



此时，打开我们的终端。找到刚才的工程的main.m,并且输入
`clang -rewrite-objc main.m`，点击回车。稍等你就会看到提示转换完成。

![转化我们的文件](http://upload-images.jianshu.io/upload_images/1835430-0056ec0cb777ee26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



这回在finder中找到工程的文件夹，在main.m同级文件夹下多了一个文件`main.cpp`，这就是转换完的文件。我们看到代码还是很多的。`直接拖到最下方`我们大概能看到点认识的了，int main。。。

这就是我们刚才`main函数里面的实现`。

![找到实现](http://upload-images.jianshu.io/upload_images/1835430-fc8be701d9be1b6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看不懂？待我帮你捋捋，去掉一些类型转换用的修饰符后剩下如下代码，是不是清晰多了？

![姑且就叫化简吧](http://upload-images.jianshu.io/upload_images/1835430-4421b3ad6ca6a611.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先看被我框选中的代码，`objc_msgSend是说发送消息`，他有`两个参数`，一个是`实例`，一个是`方法`。`objc_getClass通过字符串获取到这个类`。`sel_registerName通过字符串获取方法`。所以这句话的意思就是`给这个类发送了这个消息，消息内容就是一个方法`。

随后就容易了，给这个实例发送一个sayHello的消息，参数是后面的字符串。

通过这里我们知道我们OC的语言是怎么实现的了吧，就是`通过runtime转化成了C++的代码`，然后进行运行。

从这你也应该知道为什么OC中叫`发送消息`，不叫函数调用了吧。

另外你还应该知道为什么OC中方法`只声明`，`不实现`，`编译时只报警不报错`，`运行时crash`是为什么了吧。

既然说到这里就多少说一说C与OC吧。之前老司机说过，OC是基于C的，那么C语言中是没有对象这个概念呢，我们的对象又是什么呢？

**右手啊，程序员哪有什么对象，啧啧啧。**

除了右手，还有结构体，OC的对象就是`C语言中的结构体`。


![对象的结构](http://upload-images.jianshu.io/upload_images/1835430-b4b5b2aa725808fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到了，每个类都是一个都是一个结构体，其中有各种`指针`，指向一个类的各种`参数`，`父类、属性列表、方法列表等等`。

所以说当我们声明了类的方法，方法列表里面就有这个方法了，然后编译通过了，然后调用的时候，方法选择器去本类的方法列表里去寻找方法的实现，如果没有实现，则去其父类中寻找，如果在没有通过一系列`消息转发机制`会一直找下去，直到最后也没有找到这个方法的实现就crash了。关于消息转发，其实还有很多东西，但是在这里讲就又扯远了，所以等下期吧=。=

本例中我们用runtime做了些什么呢？

#### 使用runtime动态为category绑定属性


之前老司机说过，category是不能添加属性的。那我又要为其添加两个标识符，只能使用runtime去`动态绑定`了，在类的`属性列表`里面`通过runtime添加`上这个属性，那我就可以使用这个属性了。

首先在.h的UITableCell的category的@interface之中添加两个属性

```
@interface UITableViewCell (HeightCacheCell)
@property (assign ,nonatomic)BOOL JustForCal;//计算用的cell标识符（将计算用的cell与正常显示的cell进行区分，避免不必要的ui响应）
@property (assign ,nonatomic)BOOL NoAutoSizing;//不适用autoSizing标识符（不依靠约束计算，只进行自适应）
@end
```
这是我为期添加的两个属性，具体有什么作用，下文中会提到的，先别急。

然后在.m中添加两个属性的setter、getter方法

```
@implementation UITableViewCell (HeightCacheCell)
#pragma mark ---setter、getter---
-(BOOL)NoAutoSizing
{
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}
-(void)setNoAutoSizing:(BOOL)NoAutoSizing
{
    objc_setAssociatedObject(self, @selector(NoAutoSizing), @(NoAutoSizing), OBJC_ASSOCIATION_RETAIN);//关键字用getter的方法名，为保持关键字一致
}
-(BOOL)JustForCal
{
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}
-(void)setJustForCal:(BOOL)JustForCal
{
    objc_setAssociatedObject(self, @selector(JustForCal), @(JustForCal), OBJC_ASSOCIATION_RETAIN);
}
@end
```

这里老司机有必要说一下两个方法

- objc_setAssociatedObject
- objc_getAssociatedObject

OC自解释语言的好处就体现出来了，从函数名你就可以看出来，一个是给对象设置联系，一个是从对象获取联系。反正我英语水平就这样，我也没查字典，对不对的我就不深究了。一个setter一个getter就在这。

#### objc_setAssociatedObject

objc_setAssociatedObject(self, @selector(NoAutoSizing), @(NoAutoSizing), OBJC_ASSOCIATION_RETAIN);

方法总共四个参数，分别是`绑定目标，关键字，绑定者，策略`。

所以说简单了，绑定目标，就是给谁绑定，当然是UITableViewCell这个category了，所以self。

绑定的关键字就是说我给这个对象绑定一个属性，我总要有一个标示符去表示那个属性吧，这样我要调用这个属性的时候通过标示符去寻找才能找到这个属性。

绑定者，就是我们要为这个属性绑定的值了。

绑定策略就是说绑定的这个属性的引用机制了。这里要说明一点，`这个绑定策略如何选择，老司机目前也没有搞懂`，所以策略这里都沿用了原作者的写法，等老司机搞懂了之后会告诉大家的。


#### objc_getAssociatedObject
两个参数，一个绑定目标，一个关键字，通过关键字从绑定目标中获取属性的值。

这下是不是明白这两个setter、getter方法的意义了。

好了，这个category讲完了，他的东西真的很少。

什么，你敲不出来这两个方法？
忘了讲了，你没引入头文件。。。

```
 #import <objc/runtime.h>
```
- - -
### HeightCache

为什么说这个类呢？怎么还不进入正题呢？说好的UITableView的category呢？

因为这个类是`负责存储Cell高度`的类，而UITableView得category只是为获取cell高度提供了一个接口，当我们移动cell，添加cell，删除cell的时候要对这个高度的对应关系作出很多的操作，UITableView的category中`大量的使用了这里的方法`，所以老司机决定先把难啃的骨头解决了。

在.h中添加三个字典

```
@property (strong ,nonatomic)NSMutableDictionary * dicHeightCacheV;//竖直行高缓存字典
@property (strong ,nonatomic)NSMutableDictionary * dicHeightCacheH;//水平行高缓存字典
@property (strong ,nonatomic)NSMutableDictionary * dicHeightCurrent;//当前状态行高缓存字典（中间量）
```

为什么三个字典呢？老司机是这样考虑的，`横屏和竖屏`情况下同样内容的cell有可能是不同的，如果以同一个高度去取得话有可能出现高度不准确的问题。所以竖屏横屏分别一个字典。那这个current又是什么呢？就是`自动返回当前屏幕状态所对应的字典`那么一个中间量，这样我们写代码的时候可以不用考虑当前屏幕状态而`统一使用current这个字典`，减少很多代码量。

去.m中看看是如何实现的。


```
#pragma mark ---懒加载---
-(NSMutableDictionary *)dicHeightCacheH
{
    if (!_dicHeightCacheH) {
        _dicHeightCacheH = [NSMutableDictionary dictionary];
    }
    return _dicHeightCacheH;
}
-(NSMutableDictionary *)dicHeightCacheV
{
    if (!_dicHeightCacheV) {
        _dicHeightCacheV = [NSMutableDictionary dictionary];
    }
    return _dicHeightCacheV;
}
-(NSMutableDictionary *)dicHeightCurrent//根据系统状态返回对应字典
{
    return UIDeviceOrientationIsPortrait([UIDevice currentDevice].orientation)?self.dicHeightCacheV:self.dicHeightCacheH;
}
```

上面两个字典是懒加载，不多说了，在需要的时候创建字典。

重点是这个current。

`UIDeviceOrientationIsPortrait()`这个方法是判断括号中的状态是否是竖屏状态的一个方法，所以括号里面我们给他当前屏幕状态他就可以判断是不是竖屏了。

[UIDevice currentDevice].orientation而这句代码见名知意，取出`当前设备的屏幕状态`。

让后通过`三目运算符`返回相应的字典。

是不是写法上很简单，实际使用过程中也很方便。

上面几个工具方法最主要的主要由如下几个方法，`交换两个cell高度`的值，`插入一个cell高度`的值，`删除一个cell高度`的值。通过这三个最基本的方法组合出所有cell操作需要用到的方法。算法都很基础，没什么需要说的。

- - -
### UITableView的category

最后的主角来了。其实你会发现这里的方法并不多。因为只是向外界提供了插入，删除，删除全部，移动，计算高度五个接口。

我们看一下大体思路：


![大体思路](http://upload-images.jianshu.io/upload_images/1835430-68cc9b817178ed3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

老司机觉得自己画这图也是没谁了。。。

.h中添加一个属性，是我们刚才用来`存储高度`的那个类的一个实例。

```
@property (strong ,nonatomic)HeightCache * cache;//缓存实例
```

其实你`完全可以写在.m里当做一个私有变量去处理`，这样也`更安全`一些。
老司机写在这里是`为了调试的时候更直观`的看到缓存高度操作时的状态。实际应用中，如无特殊需要，`建议将其写在.m中`。

.m中，我们先看一下这几个工具方法，这才是核心部分。接口方法都是简单调用这几个工具方法，供外界调用的。
- - -


- -(__kindof UITableViewCell *)DW_CalculateCellWithIdentifier:(NSString *)identifier
- -(CGFloat)DW_CalculateCellHeightWithCell:(UITableViewCell *)cell
- -(CGFloat)DW_CalCulateCellWithIdentifier:(NSString *)identifier configuration:(void(^)(id cell))configuration
                           
核心算法都在这了，我一定会好好解析的。(第一个方法返回值有一个*号我敲不出来，不知道markdown什么鬼冲突。。)

#### -(__kindof UITableViewCell *)DW_CalculateCellWithIdentifier:(NSString *)identifier
第一个方法如下：

```
///从重用池中返回计算用的cell
-(__kindof UITableViewCell *)DW_CalculateCellWithIdentifier:(NSString *)identifier
{
    if (!identifier.length) {
        return nil;
    }
    NSMutableDictionary <NSString * ,UITableViewCell *> *DicForTheUniqueCalCell = objc_getAssociatedObject(self, _cmd);//利用runtime取出tableV绑定的存有cell的字典
    if (!DicForTheUniqueCalCell) {
        DicForTheUniqueCalCell = [NSMutableDictionary dictionary];//如果取不到则新建并绑定
        objc_setAssociatedObject(self, _cmd, DicForTheUniqueCalCell, OBJC_ASSOCIATION_RETAIN_NONATOMIC);//动态绑定（绑定目标，关键字，绑定者，策略）
    }
    //以上只是为了只绑定一个字典，类比懒加载
    UITableViewCell * cell = DicForTheUniqueCalCell[identifier];
    if (!cell) {
        cell = [self dequeueReusableCellWithIdentifier:identifier];//从重用池中取一个cell用来计算，必须以本方式从重用池中取，若以indexPath方式取由于-heightForRowAtIndexPath方法会造成循环。
        cell.contentView.translatesAutoresizingMaskIntoConstraints = NO;//开启约束
        cell.JustForCal = YES;//标记只用来计算
        DicForTheUniqueCalCell[identifier] = cell;
    }
    //同上，保证只有一个用来计算的cell
    return cell;
}
```

其实每一句注释都表述的很清楚。不过老司机还是会一句一句给你说的，毕竟这才是老司机的风格，恩，就是`墨迹`。


恩，老司机先说一说重用的问题吧。

我们都喜欢用tableView，因为他很好的替我们做了内存控制的问题。

他又是通过什么控制了内存呢？节省了性能呢？通过重用。

这些大家都知道。但是有很多孩子误会了重用啊，孩子你们不懂重用啊。

知道咋回事的这地方跳过吧。

前方`高能预警`，以下内容`很基础`，`真的很基础`，`真的真的很基础`，只是给一些真的不知道的人看的。

> 当一个cell将要离开屏幕时，这个cell会进入重用池。重用池并不是什么特殊的东西，就是系统给他放在一边了。他只是单纯的放在一边了，不进行任何操作。
> 
> 当一个cell将要进入屏幕的时候，会调用tableView:(UITableView *)tableView cellForRowAtIndexPath这个代理，执行其中的方法

说这两句为了说明什么呢？
第一句我想说明的是，他只是放在重用池了，`没有进行任！何！操！作！`

重点在哪呢？重点就在于存储的是整个cell，`包括cell原有的和你添加的所有子视图`。

第二句我想说明的是，他会执行代理中的`每！一！句！话！`

重点在哪呢？你从重用池中取出的cell，他是会对cell进行`再次进行绘制`。

这两句说明了什么？

- cell上不要布置太多的控件，不然存入重用池也够你吃一壶的。
- tableView:(UITableView *)tableView cellForRowAtIndexPath这里面不要添加子视图，不然下次取出同一个cell的时候还会在同样的位置再添加一个同样的子视图。你上下来回滑动的时候cell就会添加无数个子视图。
- 没有什么会影响重新绘制的，记住那句代码一定会走，只要走就一定会绘制。如果说你绘制出了什么问题，不要怪重用，跟他没关。问题一定在别的地方。

恩，这是老司机对重用的理解。

**刚才跳过的旅客朋友们，跳到这可以接上了。**

接下来我们开始说这个方法。
为什么我们要取到这个cell呢？而不是随便一个cell呢？

因为我们无法保证或者指定只使用一种cell，很多情况下我们是自定义的cell。这样的话每个不同种类的cell上的子视图是不相同的，在自动计算高度的时候对cell的布局有很高要求，所以我们一定要`保证我们计算用的cell与展示用的cell是同一种cell`。

所以说我们这个方法只有`一个参数`，identifier。因为他是从重用池中取出cell的唯一必要参数。

首先为了安全，先判断传入的identifier`是否为空`，若为空返回nil。只是为了安全。还有代码的`严谨性`。老司机又吹牛逼了，还代码严谨性，我的代码通常都考虑不周全的。。。

然后是`通过runtime从绑定的属性中取出一个字典`。如果取到的这个字典为空则创建一个字典并绑定。

为什么要创建一个字典呢，因为`我们要保证只取到这一个cell`。这个cell是为了干什么的呢，就是为了计算高度的，那么我每次计算高度的时候`只要有这么一个cell`就好了，不要去初始化太多根本不用于显示只用于计算的cell。

然后`从字典中取出我们的cell`，如果取出的cell为空，则从重用池中取出一个cell。并存入字典。

首先字典和cell的判空，都是针对第一次计算cell高度的时候来的。再次进入的时候都不会为空。
必须要说明的是，从重用池中取出cell的方法我们`一定`要使用`dequeueReusableCellWithIdentifier:`这种方式去取，不能以dequeueReusableCellWithIdentifier:indexPath那种方式去取。

就像老司机注释中说的一样，若以indexPath那种方式去取会造成鸡生蛋蛋生鸡的问题，你这程序就进入死循环了。

然后cell.contentView.translatesAutoresizingMaskIntoConstraints = NO;这句是做什么呢？事实上，有两种自动布局方式，autoResizing和autoLayout。
autoResizing是`UIView的固有属性`。是在IOS6之前用来实现自动布局的属性。当然IOS6之后的autoLayout就要比他强大不少了。

事实上这个属性`默认情况下是YES`。当为YES时，则我们`设置约束是无效`的。因为后续我们要手动`添加一个约束辅助我们计算`，所以这里我们`将其设为NO`。

然后将计算标识符置真，标识这个cell只参与计算高度，不负责展示。以后遇到批量处理cell的时候可以判断这个标识符，让其不参与运算。`当然老司机这里只是留了一个接口，实际我们有对其进行处理`。

通过这个方法，我们就成功的拿到了一个计算高度用的cell。
- - - 
#### -(CGFloat)DW_CalculateCellHeightWithCell:(UITableViewCell *)cell
第二个方法：

```
///根据cell计算cell的高度
-(CGFloat)DW_CalculateCellHeightWithCell:(UITableViewCell *)cell
{
    CGFloat width = self.bounds.size.width;
    //根据辅助视图校正width
    if (cell.accessoryView) {
        width -= cell.accessoryView.bounds.size.width + 16;
    }
    else
    {
        static const CGFloat accessoryWidth[] = {
            [UITableViewCellAccessoryNone] = 0,
            [UITableViewCellAccessoryDisclosureIndicator] = 34,
            [UITableViewCellAccessoryDetailDisclosureButton] = 68,
            [UITableViewCellAccessoryCheckmark] = 40,
            [UITableViewCellAccessoryDetailButton] = 48
        };
        width -= accessoryWidth[cell.accessoryType];
    }
    CGFloat height = 0;
    if (!cell.NoAutoSizing && width > 0) {//如果不是非自适应模式则添加约束后计算约束后高度
        NSLayoutConstraint * widthConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1.0 constant:width];//创建约束
        [cell.contentView addConstraint:widthConstraint];//添加约束
        height = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;//计算高度
        [cell.contentView removeConstraint:widthConstraint];//移除约束
    }
    if (height == 0) {//如果约束错误可能导致计算结果为零，则以自适应模式再次计算
        height = [cell sizeThatFits:CGSizeMake(width, 0)].height;
    }
    if (height == 0) {//如果计算仍然为0，则给出默认高度
        height = 44;
    }
    if (self.separatorStyle != UITableViewCellSeparatorStyleNone) {//如果不为无分割线模式则添加分割线高度
        height += 1.0 /[UIScreen mainScreen].scale;
    }
    return height;
}
```

**首先我们要想计算出我们cell的高度，就需要拿到cell的`contentView的实际宽度`。**

1.先拿到cell的宽度

2.根据辅助视图样式校正宽度


如果有自定义辅助视图，则按照自定义辅助视图的宽度去校正，如果没有按照系统辅助视图样式去校正宽度，然后`根据宽度计算高度`。

3.然后如果是`使用autoLayout进行自适应计算`。

这个也是老司机为之后留下的接口，可以控制是否进行一autoLayout进行计算。但实际并没有处理。

这里是添加约束的写法，先`添加一个宽度约束`，然后让系统`根据宽度约束自动计算高度`，接着`去掉我们添加的约束`。

4.如果根据约束计算结果错误，则`以sizeThatFits去计算高度`。

5.如果计算结果仍然为零，则给出`默认值44`。

6.判断当前tableView的分割线样式，如果有分割线，还要`校正高度`。

好了，至此你已经计算出这个cell应该有的高度了。

- - -
#### -(CGFloat)DW_CalCulateCellWithIdentifier:(NSString *)identifier configuration:(void(^)(id cell))configuration
最后一个方法：

```
///根据重用表示取出cell并操作cell后，计算高度
-(CGFloat)DW_CalCulateCellWithIdentifier:(NSString *)identifier
                           configuration:(void(^)(id cell))configuration
{
    if (!identifier.length) {
        return 0;
    }
    UITableViewCell * cell = [self DW_CalculateCellWithIdentifier:identifier];
    [cell prepareForReuse];//放回重用池
    if (configuration) {
        configuration(cell);//对cell进行操作
   }
回重用池
    return [self DW_CalculateCellHeightWithCell:cell];
}

```

这个方法就比较简单了，先判断重用标示。如果空，直接返回0。还是只为了安全。

再通过`第一个方法取出cell`，然后`将它放回重用池`，以至于下次`我们还能取出来这个cell`。

不要在意我之后还要对cell进行操作，这个`重用池只是一个概念`，其实并不是什么东西，只是标志着这里面的cell可以用于重用，你完全可以理解成他只是cell的一个标签。所以我之后还是可以继续使用这个cell。不要纠结重用池、取出、放回了，少年，他只是一个概念。

哦对了，另外有一点你要注意，你记不记得老司机说过，进入重用池，是将整个cell存储下来，`并没有做其他任何操作`。


其实你可以`重写prepareForReuse这个方法`，这里可以做任何你想做的事。比如清除所有子视图。不过有三点你需要注意：

- 别忘了先写[super prepareForReuse];
- 真清除所有子视图的时候记得别把contentView也删了
- 重写之后上面的程序中你要合理的考虑一下[cell prepareForReuse]这句话的位置，反正这么跟你讲，我是没想出来放哪。。

最后就是返回高度了。终于完事了。

- - -
### 留给外界的接口

#### -(CGFloat)DW_CalculateCellWithIdentifier:(NSString *)identifier indexPath:(NSIndexPath *)indexPath configuration:(void(^)(id cell))configuration


我只说一个，这是最重要的一个返回高度的接口了，如果每次我们都计算高度，那我们这写法也算是废了，充其量算一个自动返回高度的算法。

所以我们的逻辑应该是`先从cache里面中找，如果没有，计算并存储`。下次再找这个indexPath的时候就能找到了，正如下面的代码一样。

```
-(CGFloat)DW_CalculateCellWithIdentifier:(NSString *)identifier
                               indexPath:(NSIndexPath *)indexPath
                           configuration:(void(^)(id cell))configuration
{
    if(self.bounds.size.width != 0)//防止初始宽度为0（如autoLayout初次加载时）
    {
        if (!identifier.length || !indexPath) {//非空判断
            return 0;
        }
        NSString * key = [self.cache makeKeyWithIdentifier:identifier indexPath:indexPath];//制作key
        if ([self.cache existInCacheByKey:key]) {//如果key存在
            return [self.cache heightFromCacheWithKey:key];//从字典中取出高
        }
        CGFloat height = [self DW_CalCulateCellWithIdentifier:identifier configuration:configuration];//不存在则计算高度
        [self.cache cacheHeight:height byKey:key];//并缓存
        return height;
    }
    return 0;
}
```


- -- 
### 最后关于如何使用

就是在原来返回tableView高度的方法出调用上面那个方法，仅此而已。

特别注意，`一定要在方法中先填充数据，一定要在方法中先填充数据，一定要在方法中先填充数据`。重要的事情说三遍。否则你永远都是44啊亲们。。


![注入数据](http://upload-images.jianshu.io/upload_images/1835430-b6f493667866eee0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- - -

我知道今天这个教程看上去很抽象，所以这次我会附上demo的链接。

[demo地址](http://pan.baidu.com/s/1dFK6iHV)

不过老司机还是想说一下自己对demo这件事的看法。

老司机能选择在这里分享一些自己学到的东西，自然就不是一个敝帚自珍的人。然而之所以不爱附上demo链接是因为老司机觉得每次我都已经很详细的在博客中贴出我全部代码而且一句一句讲解真的已经知无不言言无不尽了，我觉得编程这种东西还是得下手敲一遍，看别人的东西看一天也看不懂。所以我更提倡你们自己去敲一遍。如果我把demo链接一放出来，你们直接下载了就去看，就去改，真的没有自己敲一遍学的快。当然有同学实在有需要可以留下邮箱，老司机会给你单独发demo的。
- - -

常用套话了，这么贪幕虚荣的老司机不就图你点个喜欢么=。=，觉得好点个喜欢吧。

转载记得附上链接。
http://www.jianshu.com/p/2b192257276f
