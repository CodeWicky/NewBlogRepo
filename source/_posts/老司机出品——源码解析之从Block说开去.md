
---
title: 老司机出品——源码解析之从Block说开去
layout: post
date: 2017-06-05 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- Block 
- 循环引用
categories: 源码解析
---
![从Block说开去](http://upload-images.jianshu.io/upload_images/1835430-2b26e7f92ebaaa93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

近来把《iOS与OS X多线程和内存管理》这本书又掏出来看了一遍，这本书前前后后加起来看了能有三四遍了，每次看都有新的理解。现在就把个人对Block的一些理解记录下来。

今天的内容中你会看到：

- Block是什么
- Block的实质
- 关于Block对外部的赋值操作
- Block类型及其存储域
- \_\_block说明符
- 关于Block引起的循环引用

<!-- more -->

- - -
### Block是什么？
> 带有自动变量的匿名函数。 
> 
> ————引自《iOS与OS X多线程和内存管理》

为什么这么说呢？
我们分别从匿名函数和带自动变量两个角度来说。

- 1.匿名函数

首先，Blocks是C语言的扩充功能。

C语言中函数是这个样子的：

```
void func() {
	printf("hello world");
}
```

如上，是一个C语言函数，而block是什么形式呢？

```
^void () {
	printf("hello world");
}
```
这种不带函数名的函数就是所谓的匿名函数。

- 2.带自动变量

还是要从C语言函数中说起。

```
int a = 10;
void func (int b) {
	printf("%d + %d = %d",a,b,a+b);
}

int main(int argc, char * argv[]) {
    func(5);
    return 0;
}
```
上述的代码主要想说明一件事，C语言函数中，函数体中使用的函数外部变量只有两种：函数参数即全局变量。

我们再看看Block是如何使用的。

```
int main(int argc, char * argv[]) {
    int a = 10;
    void(^block)(int) = ^(int b) {
        printf("%d + %d = %d\n",a,b,a+b);
    };
    block(5);
    return 0;
}
```
我们看到，在此例中a已经不是全局变量了，而是一个局部变量，也就是自动变量。然而block却可以正常使用，为什么呢？因为block内部维护了一个变量a的值，所以执行正确。这里你先不用纠结，下面会有源码。由上我们就知道了什么叫做带自动变量了。

- - -
### Block的实质
想要看Block的实质我们还是Block的实现过程。我们还是要借助clang。

```
#include <stdio.h>

int main(int argc, char * argv[]) {
    int a = 10;
    void(^block)(int) = ^(int b) {
        printf("%d + %d = %d\n",a,b,a+b);
    };
    block(5);
    return 0;
}
```
还是这个简单的函数，我们借助clang来转换一下。这里为了稍后方便，我们尽量删除其他无用头文件，引入必要头文件。

```
clang -rewrite-objc main.m
```

转换完成后我们会发现main.m同文件夹下多了一个main.cpp的文件。
打开这个文件我们就会看到转换后的源码，而老司机让同学们换头文件的原因你也应该明白了，引入的头文件中一些相关代码也会在转换的文件中。如果你跟老司机一样只引入了`stdio.h`的话，那么现在`command + L`跳转到第62行，复制62行至67行，`command + 下`调至文件底部粘贴，再跳至510行，`command + shift + 上`选中上面所有代码，`delete`删除后就剩下干货了，大概是这个样子的：

![block实现](http://upload-images.jianshu.io/upload_images/1835430-435ae974bab42f19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


恩，我们看到这就是block的相关实现。

首先我们看block结构体中，三个成员变量，一个构造函数。

![block](http://upload-images.jianshu.io/upload_images/1835430-72a8882606a061f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到第一个成员变量是`__block_impl`的结构体，其中有指向block实现函数的函数指针，第二个成员变量是`__main_block_desc_0`，用来负责管理block的内存管理。第三个成员变量`int型`变量a。

老司机在这里解释一下，`int a`这个成员变量就是上面提到的带有的自动变量。因为**block内部引用了外部的自动变量，所以在block结构体中多了一个同类型同名的成员变量**。同样，如果没有引入外部的自动变量的话此处block结构体中也不会有这第三个成员变量。

现在我们将目光集中到main函数中。可以看到，第一行声明了一个局域变量，第二行调用了`block的构造函数`，将block对应的`函数指针`和`Desc`以及`局部变量`传给了block。

然后我们看到第三行调用block结构体中的函数指针指向的函数，并把`block自身`及`参数`传给了函数指针指向的函数。

转过来看block指向的函数，函数中首先`从block自身中取出捕获的自动变量a复制给一个临时变量`，同时`执行原本block中的函数体`。

至此就完成了一次block的调用过程。

这里我们要注意一下捕获的自动变量：

	
所谓捕获的自动变量我们可以从两方面来理解。
	
- 1.我们看到在生成block的瞬间就将自动变量的值赋给了block。所以此时外界计时修改局部变量的值并不影响block中的值。
	
	```
	int main(int argc, char * argv[]) {
    int a = 10;
    void(^block)(int) = ^(int b) {
        printf("%d + %d = %d\n",a,b,a+b);
    };
    a = 5;
    block(5);///执行结果为15
    return 0;
}
	```
	执行结果为15，上面的话正是最好的解释。
	
- 2.block中我们是不能对捕获的变量进行赋值操作的，只要这么做编译器就会警告。为什么苹果会做出这样的限制呢？因为在block里对捕获的自动变量复制其实是有歧义的。因为通过看`__main_block_func_0`内部的实现我们知道，block内部使用的都是block捕获到自动变量，当然这个自动变量是我们转换代码之前完全不知道的一个概念。也就是在编码过程中我们在block中使用的变量与实际代码运行过程中block内部操作的变量本就是两个变量，所以在这里修改block捕获的自动变量的值事实上跟开发者预期的结果完全是两个结果。所以苹果干脆在此就给出个警告来避免未知的错误。

- 3.虽说不能对捕获的自动变量进行赋值操作，但这并`不影响我们使用他`，否则的话这个自动变量捕获到也没有什么用了。这点很好理解，没什么好解释的。

说到这里，是时候来一个本节的扣题了，所以说block的实质事实上就是一个结构体，而且是一个可以根据自身捕获的自动变量个数自动添加自身成员变量的结构体。更多情况下，其实你把它考虑成对象更好。

- - -
### 关于Block对外部的赋值操作
上文中老司机说到，Block不能对其捕获的局部（非静态）变量的值进行赋值操作。既然有这些限制，那么一定有可以Block中可以做赋值操作的变量，他们都有谁呢？

- 静态变量 
- 全局变量
- __block说明符修饰的变量



还是针对`带有自动变量的匿名函数`这句话来讲。这一节我们来探讨一下Block是如何使用外部变量的。`我们知道Block截获变量的意义在于想要使用Block作用于内无法使用的变量，所以他要截获变量。`接下来老司机围绕着这句话从各种变量类型做深入的展开。

- 1.仅使用参数的Block

```
int main(int argc, char * argv[]) {
    void(^block)(int) = ^(int a) {
        a = 10;
        printf("block : a = %d\n",a);
    };
    int a = 5;
    block(a);
    printf("a = %d",a);
    return 0;
}

///clang转换后的形式
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself, int a) {
    a = 10;
    printf("block : a = %d\n",a);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, char * argv[]) {
    void(*block)(int) = ((void (*)(int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    int a = 5;
    ((void (*)(__block_impl *, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, a);
    printf("a = %d",a);
    return 0;
}
```
由于使用的是函数的参数，是在Block作用域内可以使用的，所以Block没有对变量进行截获。这个Block基本就是最简单的函数。

- 2.使用局部变量（非静态）

```
int main(int argc, char * argv[]) {
    int a = 10;
    void(^block)() = ^() {
        printf("n = %d",a);
    };
    block();
    return 0;
}

///clang转换后
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int a;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int a = __cself->a; // bound by copy
    printf("n = %d",a);
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, char * argv[]) {
    int a = 10;
    void(*block)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    return 0;
}
```
我们看到，这里截获了这个局部变量，具体原因在上述内容中有讲到过，此处不再赘述。

- 3.局部静态变量

我们知道，静态变量存储在静态区，只创建一次，随后使用的同名变量均应指向同一地址。由静态变量的特性我们应该知道，如果Block截获了一个静态局域变量，并在Block中对其值进行了更改，这个操作应该是有效的，他应该改变该变量的值。我们看下他是如何实现的？

```
int main(int argc, char * argv[]) {
    static int a = 10;
    void(^block)() = ^() {
        a = 20;
    };
    block();
    printf("a = %d",a);
    return 0;
}
///clang转换后
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int *a;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *a = __cself->a; // bound by copy
    (*a) = 20;
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, char * argv[]) {
    static int a = 10;
    void(*block)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &a));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    printf("a = %d",a);
    return 0;
}
```
我们看到了，Block截获的是局部静态变量的指针。这个思路跟C语言中函数一样。C语言中我们想更改实参的值时也是通过传址的方式实现的。形如：

```
void mySwap(int * a,int * b);
int main(int argc, char * argv[]) {
    int a = 1;
    int b = 2;
    mySwap(&a, &b);
    printf("a = %d",a);
    return 0;
}

void mySwap(int * a,int * b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}
```

4.全局变量（静态与非静态）
老司机上面说过，Block捕获变量是为了在Block中使用其作用域外的变量，那么全局变量本身作用在区域，Block可以使用，故不需要对全局变量进行捕获。以下以全局静态变量为例。

```
static int a = 10;
int main(int argc, char * argv[]) {
    void(^block)() =  ^{
        a = 20;
    };
    block();
    printf("a = %d",a);
    return 0;
}

///clang 转换后
static int a = 10;

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
        a = 20;
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, char * argv[]) {
    void(*block)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    printf("a = %d",a);
    return 0;
}
```
- 5.\_\_block修饰的变量
我们知道，被\_\_block修饰的局部变量，在Block内部对其进行赋值操作是可以的，那么他是如何实现的呢？

```
int main(int argc, char * argv[]) {
    __block int a = 10;
    void(^block)() =  ^{
        a = 20;
    };
    block();
    printf("%d",a);
    return 0;
}
///clang 转换后
struct __Block_byref_a_0 {
    void *__isa;
    __Block_byref_a_0 *__forwarding;
    int __flags;
    int __size;
    int a;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_a_0 *a; // by ref
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags=0) : a(_a->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_a_0 *a = __cself->a; // bound by ref
    (a->__forwarding->a) = 20;
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, char * argv[]) {
    __attribute__((__blocks__(byref))) __Block_byref_a_0 a = {(void*)0,(__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0), 10};
    void(*block)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_a_0 *)&a, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    printf("%d",(a.__forwarding->a));
    return 0;
}
```

我们看到\_\_block修饰的变量会自动为其生成一个结构体，并在之后对变量的操作使用的都是结构体中持有的变量a。而后Block捕获了结构体，Block中对变量的复制也映射成了对结构体内部变量的赋值。我们可以发现，在上面的例子中，不仅生成了一个\_\_block变量的结构体，还多了`__main_block_copy_0`和`__main_block_dispose_0`两个函数，的具体作用我们稍后再表。
- - - 
### Block类型及其存储域

首先应该了解一下Block的三种类型：

- _NSConcreteStackBlock///栈区Block
- _NSConcreteMallocBlock///堆区Block
- _NSConcreteGlobalBlock///全局Block

我们设想这样一种情况，上述的例子中，我们看到我们都是在main函数中声明的Block，也就是说他其实block对象其实是一个局域变量，那么他一定会被存储在栈上。也就是说当出了变量的作用于，也就是main函数结束，block对象就会被销毁。这时我们的Block即为`_NSConcreteStackBlock`。我们可以从`__block_impl`结构体中的`isa指针`看到上述例子中的Block均为`_NSConcreteStackBlock`类型。

但是平时我们使用block的时候还有这么一种情况，即`并不`是在声明的地方`立即使用`，而是在`等待`某个时机从而进行`回调`。而此时一般是已经`出了block对象的作用域`，如果跟之前一样是`栈区block`的话显然block`已经被销毁`，此时进行`回调只会引起crash`。这时我们就需要`_NSConcreteMallocBlock`区的block了，即堆区Block。回想我们持有block的时候使用什么修饰符呢？copy对吧，而`block对象执行copy操作就是将其按需复制到堆区`。

我们看下下面的例子：

```
#import <Foundation/Foundation.h>

int globalVar = 100;

void(^globalBlock)() = ^{
    NSLog(@"global block beyond main");
};

int main(int argc, char * argv[]) {
    int a = 10;
    NSLog(@"stack block: %@",^{NSLog(@"here a = %d",a);});
    NSLog(@"malloc block: %@",[^{NSLog(@"here a = %d",a);} copy]);
    NSLog(@"global block: %@",^{NSLog(@"I use nothing");});
    NSLog(@"global block beyong main: %@",globalBlock);
    NSLog(@"global block use globalVar: %@",^{NSLog(@"%d",globalVar);});
    return 0;
}

///输出：
stack block: <__NSStackBlock__: 0x7fff5d7cd578>
malloc block: <__NSMallocBlock__: 0x60000004ef70>
global block: <__NSGlobalBlock__: 0x1024320d0>
global block beyong main: <__NSGlobalBlock__: 0x102432050>
global block use globalVar: <__NSGlobalBlock__: 0x102432110>

```

此处我们可以看到三种block类型。从源码我们可以知道默认生成的block均为`_NSConcreteStackBlock`类型，而后执行了copy操作的block为`_NSConcreteMallocBlock`类型，后面三个均为`_NSConcreteGlobalBlock`类型。

这里我们先说`_NSConcreteGlobalBlock`类型的Block。在**全局范围内声明的Block即为全局Block**，并且**没有引入自动变量的也为全局Block**。

现在我们知道了，调用过copy方法的block会被复制到堆区，堆区的Block均为`_NSConcreteMallocBlock`类型。那么什么情况下block会执行copy方法呢？

其实我们可以从上述的分析中猜到，当block需要在其作用域外使用的我们应该将其复制到堆区。例如block作为函数返回值的时候，这时候编译器会按需调用copy方法：

```
typedef void(^voidBlock)();
voidBlock func();

int main(int argc, char * argv[]) {
    NSLog(@"the return value of func:%@",func());
    return 0;
}


voidBlock func() {
    int a = 10;
    NSLog(@"the block in func:%@",^{NSLog(@"block in func : a = %d",a);});
    return ^{
        NSLog(@"block in func : a = %d",a);
    };
}

///输出：
the block in func:<__NSStackBlock__: 0x7fff56877558>
the return value of func:<__NSMallocBlock__: 0x6080000486a0>

```
此例中我们看到函数体中，**输出了一个Block其为栈区Block**，但是当将同样的**Block作为返回值返回到main函数中的时候，他变成了堆区Block**。

同学们不要说我这`不是一个Block`，我应该生成一个Block将其赋值，Log一下，在返回出去。这个真不是我不赋值，我不能啊，因为在**ARC中赋值的时候如果不附加修饰符的话默认认为生成的变量是以\_\_strong修饰符修饰的，而编译器遇到\_\_strong修饰符会自动copy**。。。我怎么给你做例子啊。。。反正老司机这么写虽然不是同一个block，但是应该是同一类型block，足以说明问题。另外老司机说过，`编译器会按需调用copy方法`。也就是说栈区block会出作用域销毁，全局block并不会，所以如果**返回值是一个全局block的话，则不会调用copy方法**。

此外以下两种情况也会由系统为我们调用copy方法：

- Cocoa框架的方法且方法名中含有usingBlock等时
- GCD的API

还有就是显示调用copy方法的时候，另外如果将其赋值给有copy修饰符修饰的属性的话也会调用copy方法。

然而什么时候应该调用copy方法呢？我们先来看下不同类型block调用copy方法会有什么行为。

> |Block类型|副本源的配置存储域|复制效果|
|:------:|:-------------:|:-----:|
| _NSConcreteStackBlock|栈|从栈复制到堆|
| _NSConcreteGlobalBlock|程序的数据区域|什么也不做|
| _NSConcreteMallocBlock|堆|引用计数增加|

> 不管Block配置在何处，用copy方法复制都不会引起任何问题。在不确定时调用copy方法即可。
> 
> ————引自《iOS与OS X多线程和内存管理》 

但是在我们确定的时候，还是要根据需要调用copy方法，不要盲目调用copy方法，毕竟这个方法是十分占用CPU资源的。

- - - 
### __block说明符

上文中，老司机已经讲述了block对象在调用copy方法时候的行为。然而__block说明符修饰的变量与block对象基本一致。

 |__block变量的配置存储域|Block从栈复制到堆时的影响|
 |:---:|:---:|
 |栈|从栈复制到堆并被Block持有|
 |堆|被Block持有|

正如老司机在上文中提到的，被__block说明符的变量会自动生成一个结构体。
值得一提的是三个地方：

![forwarding.png](http://upload-images.jianshu.io/upload_images/1835430-87eaef84e4e9496a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

老司机之前说过，只有被__block说明符修饰的变量，今后使用的均为其结构体中维护的同名成员变量，不过从源码中我们看到，并不是简单地使用了成员变量，而是`a.__forwarding->a`这样一个引用方式，这是因为什么呢？

首先从`__Block_byref_a_0`中我们可以看到__forwarding是一个`__Block_byref_a_0`类型的结构体指针。

从main函数中第一行__block变量生成的代码我们看出，在本例中生成\_\_block变量a的同时将a的\_\_forwarding指向了a自身。这样`a.__forwarding->a`最终还是指向了__block变量a结构体中的成员变量a。

既然这样，就一定存在\_\_forwarding并不指向block变量自身的情况，故此才需要\_\_forwarding存在来保证时刻能取到一个正确的值。而上文中提到的**调用copy方法的时候，就会对__forwarding指针进行操作**。
![__forwarding](http://upload-images.jianshu.io/upload_images/1835430-ee254afd55904ae5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由上图我们可以看到，当调用copy方法后，\_\_forwarding指针指向堆中的\_\_block变量。而堆中的\_\_block变量的\_\_forwarding指针则指向自身。

同时我们知道，block其实是对c语言的扩充，然而OC中我们使用的是引用计数来管理对象生命周期，而不是GC。所以事实上Block需要自行管理内存。那么当我们的Block捕获了一个对象时，他又是如何管理其引用计数的呢？
上文中老司机有提到过`__main_block_copy`和`__main_block_dispose`两个函数。**当Block结构体中捕获到的对象需要retain的时候则调用\_\_main\_block\_copy方法增加引用计数，当其需要释放的时候则调用\_\_main\_block\_dispose释放对象**。所以当block从栈上复制到堆的时候会调用copy函数，而对上的block被释放时调用dispose函数。

- - -
### 关于Block引起的循环引用

一直以来，Block引起的循环引用都让不少初级工程师，甚至包括一些中级工程师(索性就叫他中级吧。。。)谈虎色变。他们不知道Block是如何引起循环引用的，只知道`__weak可以避免循环引用`。知其然不知其所以然，闹出一些笑话也是让人无语。

首先说一下什么是循环引用？

引用计数机制不做展开，我们只需要知道，**在OC中对象是在引用计数为0的时候进行销毁的。一个对对象的强引用会造成一次引用计数的加一。释放一个强引用会造成引用计数的减一。**
![强引用](http://upload-images.jianshu.io/upload_images/1835430-e052aabc7507eaa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图我们知道，对象A对对象B有一个强引用。**当对象A销毁的时候，会释放对对象B的强引用。如果此时对象B的引用计数为0则对象B被销毁**。


![循环引用](http://upload-images.jianshu.io/upload_images/1835430-d0548d5481c4e95f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


但当出现上图的情况，强引用出现了一个`闭环`的时候，就会造成**逐个等待上一个强引用释放信号，然而闭环导致任意一个对象都不会释放对下一个对象的强引用，这就是循环引用**。

明确一点，造成循环引用的必要条件的`闭环`，所以循环引用不仅可以发生在两个对象之间，可以是多个对象，甚至可以是一个对象。

![循环引用](http://upload-images.jianshu.io/upload_images/1835430-820d88f878f0a934.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


由此看来Block引起循环引用的原因就很明白了，**Block对内部使用的自动变量造成一个强引用，而如果这个自动变量恰好对Block也有强引用的话就会造成循环引用**。

既然知道了循环引用的起因，那么我们只要打破引用的闭环就可以轻松解决。**两个思路，一个是从最开始就不让强引用成为闭环，使用弱引用。另一个思路是找到一个合适的时机主动释放一个强引用，打破闭环**。

![打破闭环](http://upload-images.jianshu.io/upload_images/1835430-b5663570b518b542.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 弱引用

```
__weak typeof(self)weakSelf = self;
self.block = ^{
    NSLog(@"%@",weakSelf);
};
self.block();
```
上述代码中，使用__weak生成一个弱引用变量weakSelf，保持对self的弱引用。然后Block捕获到weakSelf，对weakSelf也是弱引用，然而却没有造成闭环。故避免了循环引用。
![weakSelf](http://upload-images.jianshu.io/upload_images/1835430-2028bfe9fbf497e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 主动释放

```
__block id blockSelf = self;
self.block = ^{
    NSLog(@"%@",blockSelf);
    blockSelf = nil;
};
NSLog(@"%@",self.block);
self.block();
```
上述代码中，使用\_\_block生成一个block对象blockSelf，保持对self的强引用。然后Block捕获到blockSelf，强引用blockSelf，由于self对block还有一个强引用，此时形成了一个闭环。但当block调用的时候，内部最后将blockSelf对象置为nil。由于blockSelf置为nil，__block对象失去强引用被销毁，同时释放对self的强引用，从而打破闭环。
![blockSelf](http://upload-images.jianshu.io/upload_images/1835430-1c2640ded7e6ad98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不过两种避免循环引用的方式都有各自的缺点。

\_\_weak 的弱引用形式的缺点在于，当block执行的时候，由于对self是弱引用，不能保证self对象是否已经被销毁。事实上block执行前self被销毁还好，顶多是不执行。但是如果在block执行过程中，self被销毁就会造成不可预估的后果。所以当使用\_\_weak的时候我们通常会看到如下结构：

```
__weak typeof(self)weakSelf = self;
self.block = ^{
	__strong typeof(weakSelf)strongSelf = weakSelf;
    NSLog(@"%@",strongSelf);
};
self.block();
```
这样的结构可以保证在block执行过程中，不会因为self释放引起问题，然而如果block执行前self被释放后block也就没有机会执行了，也算是对代码的保护。更多的关于`Weak-Strong-Dance`的讨论可以看下这篇文章：

> [Weak-Strong-Dance真的安全吗？](http://www.jianshu.com/p/737999a30544)
 
\_\_weak有这样的缺点，为什么不适用\_\_block等方式呢？

事实上\_\_block同样有着自己的烦恼，就是一定要在block体中对\_\_block对象置为nil，且block一定要执行才可以解决循环引用。所以开发者要根据具体情况合理的选择解决循环引用的方式。
- - - 
至此，老司机今天的内容也就算结束了。

参考资料：
- 《iOS与OS X多线程和内存管理》
- [Weak-Strong-Dance真的安全吗？](http://www.jianshu.com/p/737999a30544)
