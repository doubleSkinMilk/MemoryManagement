# Effective Objective-C 2.0 编写高质量iOS与OS X代码的52个有效方法
[TOC]
##(1)了解objective-C语言的起源
>oc使用消息结构而非函数调用
>oc是由SmallTalk演化来的,他是消息型语言的鼻祖

```
oc是这样:
object *obj = [object new];
[obj someFunc];

c++是这样
object *obj = new object;
obj->someFunc();
```
>消息结构的语言,其运行时所执行的代码由运行环境来决定,而函数调用的语言,有编译器来决定
>采用消息结构的语言,总是在运行时才回去查找所执行的方法,实际上编译器甚至不关心接收消息的对象是何种类型,接收消息的对象也要在运行时处理,这个过程称之为`动态绑定`

>oc的很多工作都是由`运行期组件(runtime component)`来完成的,而非编译器,例如runtime包含全部的内存管理方法

>oc是c语言的超集,所以c语言的所有功能在oc中都适用.因此掌握c语言是很重要的,其中c语言的内存模型一定要理解,这有助于理解oc的内存模型以及`引用计数`机制的工作原理


```
NSString *str = @"someString";
NSString *anotherStr = str;
```
>此变量指向NSString的指针,str分配在`栈`,someString这个对象分配在`堆空间`,也就是说实际上str这个指针指向someString堆空间的地址值;
str和anotherStr这两个变量在当前`栈帧`中分配了两块内存,每块内存都能容下一枚指针,这两块内存的值都一样,即someString堆空间的地址值

>oc分配在堆空间的内存必须直接管理,栈中会在栈帧弹出时自动清理;
>oc将内存管理抽象出来了,不用malloc和free,取而代之的是oc的运行时,它把这部分工作抽象成一套内存管理架构,叫做`引用计数`
>

__oc还有不带*的变量,他们可能会使用`栈空间`__

```
struct CGRect{
	CGPoint origin;
	CGSize size;
}

CGRect frame;
frame.origin.x = 0.0f
```
>其实整个系统框架都使用结构体,如果改用oc对象来做 性能会受影响,还会造成额外的额内存开销
>这些用来保存基础数据类型,而非对象类型,用结构体就可以了

1:的oc为c语言添加了面向对象的特性,是其超集;oc使用了动态绑定的消息结构,即在运行时才会检查对象类型.接收到一条消息后,究竟应执行哪段代码,由运行期环境而非编译器决定
2:理解c语言的核心概念有助于写好oc程序,尤其要掌握内存模型和指针

##(2)在类的头文件中尽量少引用其他头文件
>oc编写类的标准方式为:分别创建两个文件,头文件.h 实现文件.m

定义一个`班级`看起来就是这样

```
#import <Foundation/Foundation.h>
@interface Classes:NSObject
@end
#import "Classes.h"
@implementation Classes
@end
```
>在oc中 几乎任何类都要引入`Foundation`框架

让我们来创建一个`学生类`


```
#import <Foundation/Foundation.h>
@interface Student:NSObject
@end
#import "Student.h"
@implementation Student
@end
```
现在要在班级类加一个学生

```
@interface Classes:NSObject
@protocol (nonatomic,strong) Student *someStudent;
@end
```
这时候Classes不知道有Student,所以要导入头文件

```objc
#import "Student.h"
@interface Classes:NSObject
@protocol (nonatomic,strong) Student *someStudent;
@end
```
但是这样做的话,会一并将Student的全部实现导入到Classes中,这显然不够优雅,这个时候应该使用关键字`@class`


```objc
@class Student
@interface Classes:NSObject
@protocol (nonatomic,strong) Student *someStudent;
@end
```
>@class这个关键字表示`"向前声明(forward declaring)"`
仅仅表示有个类名叫Student
若要使用student,需要在在.m文件中#import "Student.h" 使当前类知道student的接口细节

__这样做的好处是将引入头文件的时机尽量延后,只有需要时才引用,可以减少类使用者否文件数量,试想如果还有一个类需要引入classes头文件 会将student的内容一并加进去,但是这个类根本用不到student,这样不仅逻辑混乱而且会增加编译时间__

>有时候student头文件需要引用classes,同时classes头文件也要引用student,这时会产生`循环引用`,解决方法是一方用@class 就可以打破循环引用

1:除非确有必要,否则不要引入头文件;一般来说,应在某个类的头文件中使用向前声明来提及别的类,并在实现文件中引用那个类的头文件,这样做可以尽量降低类之间的耦合


##(3)多用字面量语法,少用与之等价的方法
>NSString,NSNumber,NSArray,NSDictionary隶属于Foundation框架
>他们有初始化方法,比如先alloc在init
>`NSString *str = [[NSString alloc]initWithString:@"someString"];`
这样做太长了,还麻烦,不过在objective-c 1.0之后就可以用字面量语法了 
`NSString *str = @"someString";`
使用字面量语法,可以所见源代码的长度,使其更为易读

__(3.1)NSNumber__
>把整数,浮点数,布尔值封入NSNumber中

```objc
 NSNumber *intValue = @1;
 NSNumber *floatValue = @1.2f;
 NSNumber *doubleValue = @1.22344567;
 NSNumber *boolValue = @YES;
 NSNumber *charValue = @'a';
```
__(3.2)NSArray__
>可以用字面量初始化数组,也可以用下标访问数组

```
NSArray *array = @[@"dog",@"cat"];
array[0];//dog
NSArray *array = [NSArray arrayWithObjects:@"dog",nil,@"cat", nil];
//这样初始化结果是array只包含一个dog因为遇到nil方法就会提前结束
//相反用字面量就不会有这个问题,因为遇到nil就直接报异常了crash掉
```
__(3.3)NSDictionary__
>系统api初始化是<对象>,<键>,<对象>,<键>非常不易理解

```
NSDictionary *dict = [NSDictionary dictionaryWithObjectsAndKeys:@"even",@"name",@12,@"age", nil];
//不易理解
NSDictionary *dict = @{@"name":@"even",@"age":@12};
//用字面量就会清晰好多
NSString *name = dict2[@"name"];
//访问特定键值
```
__(3.3)可变数组与可变字典__
>字面量初始化方法生成的是不可变对象,若要可变对象需要mutableCopy一份

```
NSMutableArray *array = [@[@"dog",@"cat"] mutableCopy];    
NSMutableDictionary *dict = [@{@"name":@"jack"} mutableCopy];
```

>可以直接对下标进行修改操作

```
mutableArray[0] = @"changeValue";
mutableDict[@"name"] = @"Rose";
mutableDict[@"adress"] = @"xxxxx";
```

1:尽量使用字面量语法初始化字符串,数组,字典,数值
2:通过下标操作来访问对应的元素
3:用字面量语法初始化时,若值是nil,则会抛出异常,因此务必保证值不为nil


##(4)多用类型常量,少用#define预处理指令


```objc
#define DURATION 0.3
```
>\#define 当我们想定义一个持续时间的常量DURATION 将其设置成0.3;这里会将所有DURATION替换成0.3,从而缺失了类型检查的过程

```objc
static const NSTimeInterval kDURATION = 0.3;
```
>static const这种定义方式,包含了类型信息,可以清楚的描述常量的含义,坑有助于编写开发文档
>注:在.m实现文件中 变量名前面加k,在.h中通常加类名为前缀,还有在头文件定义常量是很糟糕的事情,因为oc没有命名空间
>static表示kDURATION只在当前.m文件中(编译单元)可见,const表示kDURATION不可修改
>如果不加static那么编译器会生成一个`外部符号(external symbol)`,此时如果在另一个编译单元中声明了同名的kDURATION就会报错:duplicate symbol duplicate in xxx.m,yyy.m
>事实上,同时声明static和const,那么编译器根本不会创建符号,而是像预处理命令一样,把所有kDURATION替换为0.3,只不过多了类型检查

___有时,需要定义一个公开的常量___
>例如UIKit的通知

```
//头文件
extern NSString *const UIApplicationDidBecomeActiveNotification;

//实现文件
NSString *const UIApplicationDidBecomeActiveNotification = @"UIApplicationDidBecomeActiveNotification"
```
>以上声明之后,在全局符号表中就会有名叫UIApplicationDidBecomeActiveNotification的符号,编译器无序查看其定义,即允许代码使用此常量,因为在全局符号表里,所以命名要谨慎
>此类常量必须要定义,而且只能定义一次

这样定义的常量要优于define,因为编译器会确保常量值不变,一旦在.m中定义好 随处都可以使用,也不用担心常量被人修改

1:不要用define来定义常量,这样做不包含类型信息,编译器只会进行替换操作,即使有人重新定义了常量值,编译器也不会发出警告,会导致程序中常量值不一致
2:在.m文件中使用static const来定义`只在编译单元内可见的常量`不会在全局符号表中,因此无需为其加命名前缀
3:在头文件使用extern来声明全局常量,并在实现文件中为其定义值,这种常量会出现在全局符号表中,所以名称应该加上类的前缀,加以区隔

