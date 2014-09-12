---
layout: post
title: "Understanding Infix Notation in OC"
date: 2014-09-12 17:19:20 +0800
comments: true
categories: Objective-C 中缀符
---

# 理解 Objective-C 中缀符

----------

初学 OC ，对其方法的声明语法感到很奇怪。比如：

声明：

```c
@interface C : NSObject
+(int) fun: (int) a second: (int) b;
```

定义：

```c
@implementation C
+(int) fun: (int) a second: (int) b
{
    return a * b;
}
@end
```

调用：

```c
int a = [C fun: 2 second: 3];
```

书上说这种语法称为中缀符的形式，我的疑问是：

> * `fun` 应该就是方法名吧？那 `second` 理解为什么？
> * 如果 `second` 理解为第二个参数的名字，那 `b` 又是什么？同时 `fun` 又是什么？

好吧我的问题也许有些奇怪，对于一个习惯了 C/C++ 等“正常”语法的人来说，一时没转过弯来。一番 gb 之后，参考了多人的说法，算是基本理解了 OC 的方法参数名。

整理如下：

> * **方法修饰符**
  `-` 代表此方法是`实例方法`，必须先生成类实例，通过实例才能调用该方法
  `+` 代表此方法是类的`静态方法`，可以直接调用，而不用生成类实例
> * **参数类型**
  a 与 b 分别是两个参数，均为 int 类型
> * **方法签名**
  本例中，fun 和 second 组成了该方法的签名关键字
  不过还是有些怪异，这样理解，第一个参数是没有参数名的，**如果硬要说有，那就是方法名**，统一说来，见到冒号，冒号前面那个就是参数名

再举例，也可以编写没有参数名的方法定义与调用：

```c
+(int) fun: (int) a: (int) b;
int a = [C fun: 2: 3];
```

