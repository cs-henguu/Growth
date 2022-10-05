## Unity多线程

随机出现的bug一般与线程调度有关。

应用场景：

- 大量耗时的数据计算；
- 网络请求；
- **复杂密集的文件I/O 操作**；

**注意到：Unity是单线程设计的游戏引擎，子线程中是无法运行UnitySDK的。**

因此，网络回调函数里也不能生成一个游戏物体，因为网络回调函数默认是子线程，而生成游戏物体是UnitySDK的操作。

### Unity主线程  Unity生命周期

Unity的主流程生命周期函数就是主线程，常见的生命周期函数：

`Awake()`：程序一开始运行就执行一次，只执行一次；

`OnEnable()`：启用事件，只执行一次，当脚本组件被启用时执行一次；

`Start()`：开始事件，执行一次；

`FixedUpdate()`：固定更新事件，执行N次，0.02秒执行一次。所有物理组件相关的更新都在这个事件中处理；

`Update()`：更新事件，执行N次，每帧执行一次；

`LateUpdate()`：稍后更新事件，执行N次，在`Update()`执行完毕后再执行；

`OnGUI()`：GUI渲染事件，执行N次，执行的次数是`Update()`事件的两倍；

`OnDisable()`：禁用事件，执行一次，在`OnDestroy()`之前执行，或者当脚本被禁用后也会触发该事件

`OnDestroy()`：销毁事件，执行一次。脚本挂在的gameobject被销毁时执行。

### 子线程

不在主线程运行的程序都叫子线程。

```c#
using System.Collections;
using System.Collections.Generic;
using System.Threading;
using UnityEngine;
 
public class NewBehaviourScript : MonoBehaviour {
    private Thread T;
 
    void Start () 
    {
        T = new Thread(ThreadTest);	// 创建一个新线程
        T.Start();
	}
	
	void ThreadTest()
    {
        for(int i = 0; i < 3; i++)
        {
            Debug.Log("子线程运行了");
        }
    }
}
```



## Unity协程

协程不是子线程，对于Unity主线程的设计，因此更倾向于使用time slicing**时间分片**的协程(*coroutine*)去完成异步任务，融合到Unity的生命周期中。

协程是编译器级别的，本质上还是一个线程的时间分片去执行代码段，使代码段可以分段式运行，显示调用`yield()`才被挂起，有一个类似栈的数据结构保存现场。

Unity中，协程是可以自行停止运行`yield` ,直到给定的 *yieldInstruction*结束再继续运行的函数。因此可以将代码段分散到不同的帧中，每次执行一段，下一帧再执行`yield`挂起的地方。

## Unity的各个模块



## Unity碰撞器与触发器

## FixedUpdate和Update

## ECS

Entity-Component-System 实体-组件-系统 一种软件架构模式，主要用于游戏开发，遵循组合优于继承的原则，每个实体不是由类型Class定义的，而是由与其关联的组件定义的，组件如何与实体关联，取决于ECS框架如何设计。

> Entity 实体：本质上是存放组件的容器
>
> Component 组件：游戏所需的所有数据结构
>
> System 系统：根据组件数据处理逻辑状态的管理器

与面向对象的区别：代码是基于组件来操作而非基于对象。

设计一个玩家使用武器攻击其他玩家造成伤害的模式：

- OOP 面向对象的思维是：总共有哪几类对象，对象的行为是什么。需要玩家对象和武器对象，玩家能攻击也能被攻击，能使用武器，有血量属性。武器对象有攻击力和所有者等属性。
- ECS 基于组件的思维是：有哪些行为，行为对应的数据是什么。有攻击和受伤害两种行为，攻击行为只关注攻击力，受伤害行为关注血量和所受伤害。于是设计伤害系统，武器组件，血量组件和碰撞组件。如果实体能攻击，就给他添加武器组件，能被攻击就添加血量组件。哪个实体被攻击，就加上碰撞组件。

参考资料：[1](https://www.bilibili.com/read/cv16047480) [2](https://zhuanlan.zhihu.com/p/41652478) [3](https://zhuanlan.zhihu.com/p/138029194)

## DOTS

Unity官方的ECS框架，全称为Data-Oriented-TechStack 多线程式数据导向型技术堆栈

DOTS包括三个部分：

- ECS：提供默认情况下写出高性能的代码的方法
- JobSystem：Unity的多线程解决方案，提供编写安全简单的多线程代码的方法
- Brust：一种基于LLVM的后端编译技术，将C#编译为高度优化的本机代码

DOTS为什么会快？ 因为catch命中率高，且数据在JobSystem中以多线程方式处理，DOTS保证相同类型的组件在内存中都是顺序排列，从而极大增加了catch命中率。

## Gameplay框架

GameObject + Monobehavior



## 高性能C#

C#语言无法控制数据在内存中的分布，标准库面向的是“堆上的对象”和“具有其他对象指针引用的对象”，当处理性能敏感的代码时，可以**放弃使用大部分标准库**，例如 **Linq、StringFormatter、List、Dictionary**。禁止内存分配，即不适用类，**只使用结构、映射、垃圾回收器和虚拟调用**，添加可使用的部分新容器如`NativeArray<T>`

大部分的原始类型(float, int, uint, short, bool), enums, struct 和其他类型的指针

集合：用NativeArray\<T> 代替T[]

所有的控制流语句（除了try, finally, foreach, using）

对throw new XXXEception(...)给与支持

## JobSystem

多线程要保证线程安全，竞态条件也就是计算结果依赖于两个或更多进程被调度的顺序。低效的多线程之间上下文切换十分耗时。

数据冲突，不确定性和死锁是主要挑战。

Unity的特性是确保代码调用的函数和所有内容不会在全局状态下读取和写入，

## Burst

Unity构建了名为 Burst 的代码生成器和编译器。

当使用C#时，对整个流程有完整的控制，包括从源代码编译到机器代码生成。

程序员可以把C++语言的性能敏感代码移植为HPC#(高性能C#)

## 动画系统

Mecanim

Unity中的动画系统包括 可重定向动画、运行时对动画权重的完全控制、动画播放中的事件调用、复杂的状态机层级视图和过渡、面部动画的混合形状等。

**基于属性绑定、关键帧、按时间在关键帧之间插值的原理来实现的。**



# 反射机制总览

[UE4反射](https://zhuanlan.zhihu.com/p/60622181)

反射在C#等语言比较常见，反射数据描述了类在运行时的内容，反射数据存储的信息包括类的**名称**、类中的**数据成员**、每个数据**成员类型**、每个成员位于对象**内存**映像的**偏移***offset* 以及类的所有**成员函数**信息。

C++本身不支持反射，UE4在C++基础上搭建了一套自己的反射机制。

具体来说，对于一个类(UClass)，可以获得这个类的所有属性和方法；对于一个类对象(UObject    )，如果其方法和属性被纳入UE4的反射机制，那么可以调用它所拥有的方法和属性。所以，UE4能够通过反射机制实现序列化、editor的details panel、垃圾回收、网络复制、蓝图/C++通信和相互调用等功能。

只有主动标记的类型、属性、方法会被反射机制追踪，**UnrealHeaderTool（UHT）**会收集这些信息，生成用于支持反射机制的C++代码，然后编译工程。

## Java的反射机制

在运行状态中，对于任意一个类Class文件，都能知道这个类的属性和方法；对于任意一个对象，都能调用它的方法和属性；这种动态获取的信息以及调用对象的方法的功能成为java语言的反射机制。

简单来说，**动态获取类中的信息以及动态调用对象的方法和属性的功能，就是Java的反射机制。**通过反射机制，类就是透明的，可以获取这个类的任意东西。

### 反射机制的使用

获取**字节码**文件的三种方法：[java反射](https://blog.csdn.net/nanZhaiXiaoLang/article/details/113876009?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-113876009-blog-79407428.pc_relevant_blogantidownloadv1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-113876009-blog-79407428.pc_relevant_blogantidownloadv1&utm_relevant_index=2)

### 应用实例

## UE4反射机制的原理

UObject是反射机制的核心。每一个继承UObject且支持反射机制的类型都有一个相应的UClass或者它的子类，UClass包含了该类的描述信息。UObject与UClass组成了UE4对象系统的基石。

### 标记

标记一个包含反射类型的头文件，需要添加一个include 通知UHL处理。

```c++
#include "FileName.generated.h"
```

可以使用 UENUM() UCLASS() USTRUCT() UFUNCTION() UPROPERTY()来标记不同的类和成员变量，标记也可以包含额外的描述关键字。

也可以声明非反射类型的属性，这些属性对反射机制是不可见的（存储一个UObject引用的裸指针会导致垃圾回收系统无法获取到该引用，造成内存泄露）

每一个描述关键字例如EditAnywhere 或 BlueprintCallable 都在ObjectMacros.h中有一个镜像，当不知道关键字的意思时，可以去ObjectMacros.h中查看。

### UnhealHeaderTool 和 UnrealBuildTool

反射C++代码通过UnhealHeaderTool 和 UnrealBuildTool产生的，UBT扫描头文件，记录所有包含反射类型的modules，当有头文件改变时，用UHT更新反射数据。UHT解析头文件，扫描标记，生成用于支持反射的C++代码。

UHT被设计为一个独立程序，自身不使用任何的generated headers。