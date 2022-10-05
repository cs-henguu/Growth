# C++

## 内存管理

堆能分配较大内存，使用malloc和free

栈为函数的局部变量分配内存

全局存储区，存储全局或静态变量

自由存储区：通过new和deleta分配和释放内存，具体实现可能是堆或者内存池

### 深拷贝与浅拷贝

有指针成员时，要用深拷贝，否则容易导致悬挂指针的问题。

深拷贝会申请堆内存的空间来存储数据，拷贝会另外创造一个一模一样的对象，新对象跟原对象不共享内存，修改新对象不会改到原对象。

浅拷贝只复制指向某个对象的指针，而不复制对象本身，新旧对象还是共享同一块内存。

递归方法实现深拷贝原理：遍历对象、数组直到里边都是基本数据类型，然后再去复制，就是深度拷贝

### new与malloc的区别

使用上：

new和delete是关键字，malloc和free是库函数；

new可以为自定义的对象类型申请内存空间，返回的是一个指向对象的指针，失败则报错；malloc则需要指定申请的内存空间大小，成功则返回一个void型指针，失败返回null

原理上：

new先执行`operate_new`申请内存空间，再执行构造函数，其中`operate_new`的是用malloc实现的；

delete先执行析构函数，再执行`operate_delete`释放内存空间，`operate_delete`是用free实现的；

### 堆栈

堆和栈之间并没有一个固定的界限。

在程序编译链接生成可执行程序之后，堆和栈的起始地址就已经固定了，但是堆是向高地址增长，栈是向低地址增长，堆和栈共用全部的自由空间，只不过各自的起始地址和增长方向不同。

如果在堆和栈增长到相互覆盖时，会产生**“堆栈冲突”**，这种错误只能靠设计者自己解决。

### 智能指针

- 三种智能指针及其原理，区别，用在什么地方

- shared_ptr怎么实现多指针指向同一个地址

- make_shared 和 new shared_ptr的区别

  make_shared内部用move语义实现，而new分配两次内存，make_ptr分配一次

- 引用计数如何保证不同实例的指针之间共享同步

  引入信号量，保证一次只有一个实例能够修改count

- 怎么解决循环引用

- weak_ptr的代码实现

- 智能指针构造与析构时间

  一般构造和复制构造两种，析构是每次执行delete判断--count是否为0，为0则执行析构

- 野指针的产生原因

- unique_ptr的作用

原始指针容易造成悬挂指针或者野指针。

```c++
objtype *p = new objtype();
p->func();
delete p;
```

上面的代码定义了指向`objtype`的指针`p`，`p`执行`objtype`类的成员函数`func()`，一般会犯的错误是没有及时`delete p;`。而有的时候，即使及时`delete`，仍然有可能在执行`func()`的时候提前终止，使得无法执行到后面的`delete p;`。因此需要智能指针。

实现一个shared_ptr：

```c++
#include<iostream>
#include<string>
#include<map>
#include<unordered_map>
using namespace std;

template <typename T>
class Shared_Ptr
{
private:
    T* ptr;
    unsigned int * count;
public:
    Shared_Ptr(T *_ptr)
        :ptr(_ptr){
    	count = new int(1);        
    }
    Shared_Ptr(Shared_Ptr<T>& sp)
        :ptr(sp.ptr)
        ,count(sp.count){
        (*count)++; 
    }
    Shared_Ptr<T>& operator=(const Shared_Ptr<T>& sp){
        if(sp.ptr != ptr){
            if(--(*count)==0){
            	delete ptr;	delete count;
            	ptr = nullptr; count = nullptr;
        	}
            ptr = sp.ptr;
            count = sp.count;
            (*count)++;
        }
        return *this;
    }
    ~Shared_Ptr(){
        if(--(*count)==0){
            delete ptr;	delete count;
            ptr = nullptr; count = nullptr;
        }
    }
    T& operator*(){
        return *ptr;
    }
    unsigned int Use_count(){
        return *count;
    }
};
```



## STL

- 实现stack的push函数

vector  变长数组，实现机制是一块儿连续内存的数组，按当前存储大小的两倍来开辟空间，用三个指针来管理，begin end 和 指向内存末端的end。vector可以随机访问，迭代器访问复杂度O1，查找的话如果是无序是ON，有序用二分查找是OlogN，插入和删除看位置，需要整体移动，因此尾部最容易插入删除。

list 双向循环链表，插入删除比vector简单，不能随机访问，只能扫描查找复杂度ON

set和map  红黑树实现 增删改查复杂度接近OlogN

### vector扩容

vector的内存空间只会增长，不会减少。vector内的元素连续存放，vector有一个capacity()函数用来获取当前的vector的剩余预留空间。每当vector需要扩容时，都会对当前容量加倍扩容并重新分配vector的内存空间。

vector的删除操作并不会使vector的占用空间发生改变，即使clear()也不会。如果需要动态缩小空间，建议使用另一个容器 deque。



### 迭代器失效的情况

- 对于vector：由于vector是连续内存，所以insert和erase操作之后，都会导致插入点和删除点之后的元素移动位置，插入点和删除点之后的迭代器会失效。解决办法是重新给迭代器赋值`iter = vec.erase(iter);` erase会返回下一个有效迭代器的值，insert同理。
- 对于list：删除操作使指向被删除位置的迭代器失效，其他迭代器不变，因为list是不连续分配的内存。解决办法：`erase(iter++)`或者`iter = erase(iter);`插入并不会出现迭代器失效。
- 对于map set等树形结构：树形结构的插入不会使迭代器失效，删除会。解决办法: `erase(iter++);` erase()返回void，所以只能通过这个办法避免失效。

### 删除vector index处的元素

可以先将vector的末尾元素覆盖index处的元素，再执行pop_back()

## 关键字

### inline函数的作用

编译器会将inline函数直接嵌入到调用语句处，而不是生成函数调用语句。

优点是：执行速度快

缺点是：复制代码段，浪费内存空间

类内成员函数隐式为内联函数，建议将逻辑简单的函数定义在类内，而涉及到递归循环的函数定义在类外的普通函数，如果内联函数中有循环递归，则会不停地复制代码段。

## 指针

### 指针和引用的区别

C++primer中的对象是指 一块能存储数据并拥有某种类型的内存空间，一个对象有值和地址，运行程序时，计算机会为该对象分配空间存储值，并可以通过内存地址访问到值。

指针本质上依然是对象，只不过指针的值是其他对象的地址，通过指针就可以访问到其他对象。

而引用是变量的别名，定义一个引用的时候，程序把该引用和它的初始值绑定在一起，而不是另外拷贝一次初始值。**引用必须在声明的时候就初始化。**这也是类内初始化成员列表的存在意义之一。**引用一经声明就不可以绑定到其他对象。**相对于指针，引用由编译器保证初始化，一定不为空，因此不需要检查是否为空，提高了效率。

更深层次的讨论，引用和指针的汇编代码一模一样，引用其实就是C++的语法糖，可以看作编译器自动完成取地址、解引用的常量指针。

## 多态

一个接口多种行为或者说函数名一样但是状态不一样。

通过重载，继承或者虚函数实现

### 类

#### 构造函数

构造函数：函数名与类名相同，对象实例化时编译器自动调用构造函数，构造函数可以重载，无返回值；

默认构造函数：没有参数的构造函数，或者每个参数都有初始值。

```c++
class Complex 
{         
private :
    double m_real;
    double m_imag;
 
public:
    // 无参构造函数
    // 如果创建一个类你没有写任何构造函数,则系统会自动生成默认的无参构造函数，函数为空
    // 只要你写了一个下面的某一种构造函数，系统就不会再自动生成这样一个
    //默认构造函数，如果希望有一个这样的无参构造函数，则需要自己显式地写出来
    Complex(void)
    {
         m_real = 0.0;
         m_imag = 0.0;
    } 
         
    // 一般构造函数
    // 一般构造函数可以有各种参数形式,一个类可以有多个一般构造函数，前提
    //是参数的个数或者类型不同（基于c++的重载函数原理）
    // 例如：你还可以写一个 Complex( int num)的构造函数出来
    // 创建对象时根据传入的参数不同调用不同的构造函数
    Complex(double real, double imag)
    {
         m_real = real;
         m_imag = imag;         
     }
     
    // 复制构造函数（拷贝构造函数）
    // 复制构造函数参数为类对象本身的引用，用于根据一个已存在的对象
    //复制出一个新的该类的对象，一般在函数中会将已存在对象的数据成员的值复制一份到新创建的对象中
    // 若没有显示的写复制构造函数，则系统会默认创建一个复制构造函数，但当类中
    //有指针成员时，由系统默认创建该复制构造函数会存在风险
    Complex(const Complex & c)
    {
        // 将对象c中的数据成员值复制过来
        m_real = c.m_real;
        m_img  = c.m_img;
    }            
 
    // 类型转换构造函数，根据一个指定的类型的对象创建一个本类的对象
    // 例如：下面将根据一个double类型的对象创建了一个Complex对象
    Complex::Complex(double r)
    {
        m_real = r;
        m_imag = 0.0;
    }
 
    // 等号运算符重载
    // 注意，这个类似复制构造函数，将=右边的本类对象的值复制给等号
    //左边的对象，它不属于构造函数，等号左右两边的对象必须已经被创建
    // 若没有显示的写=运算符重载，则系统也会创建一个默认的=运算符重载，只做一些基本的拷贝工作
    Complex &operator=(const Complex &rhs)
    {
        // 首先检测等号右边的是否就是左边的对象本，若是本对象本身,则直接返回
        if ( this == &rhs ) 
        {
            return *this;
        }
             
        // 复制等号右边的成员到左边的对象中
        this->m_real = rhs.m_real;
        this->m_imag = rhs.m_imag;
             
        // 把等号左边的对象再次传出
        // 目的是为了支持连等 eg:    a=b=c 系统首先运行 b=c
        // 然后运行 a= ( b=c的返回值,这里应该是复制c值后的b对象)    
        return *this;
    }
};

void main()
{
    // 调用了无参构造函数，数据成员初值被赋为0.0
    Complex c1，c2;
 
    // 调用一般构造函数，数据成员初值被赋为指定值
    Complex c3(1.0,2.5);
    // 也可以使用下面的形式
    Complex c3 = Complex(1.0,2.5);
         
    // 把c3的数据成员的值赋值给c1
    // 由于c1已经事先被创建，故此处不会调用任何构造函数
    // 只会调用 = 号运算符重载函数
    c1 = c3;
         
    // 调用类型转换构造函数
    // 系统首先调用类型转换构造函数，将5.2创建为一个本类的
    //临时对象，然后调用等号运算符重载，将该临时对象赋值给c1
    c2 = 5.2;
       
    // 调用拷贝构造函数( 有下面两种调用方式) 
    Complex c5(c2);
    Complex c4 = c2;  // 注意和 = 运算符重载区分,这里等号左边的对象不
                                  //是事先已经创建的，故需要调用拷贝构造函数，参数为c2       
   
}

```



#### 拷贝构造函数与移动构造函数的区别

拷贝构造函数的参数类型是对一个对象的左值引用，通过一个对象去初始化另一个对象，是两个对象之间的复制操作。如果传入的参数是一个临时对象，如果使用拷贝构造函数则需要先初始化该临时对象，为该临时对象开辟一份内存空间，再开一份新的另外的内存空间初始化一个对象，把临时对象的成员复制过来。最后再释放临时对象的内存。

移动构造函数的参数类型是对一个对象的右值引用，使用右值引用的好处是可以将一个临时对象的内存转移给后续对象，减少不必要的内存分配和释放。

#### 左值引用与右值引用

右值引用 && 只能绑定临时对象

左值引用 & 

#### 右值引用与move()

move()用来将一个左值引用转换为右值引用，可以将其看成一个对左值的强制类型转换`static_cast<T&&>(left_value);`

通常move()用来和移动构造函数结合使用，初始化对象。

#### 引用折叠是什么

(https://blog.csdn.net/kupepoem/article/details/119944958)

将多个的&&&折叠为1个或2个

- 对类取sizeof()的结果

- 与结构体的区别

- 结构体里定义一个虚函数，大小怎么变化

- 结构体增加一个static int，大小怎么变化

- 空类

- 重载运算符的方法

- inline函数的作用

- 类内的静态变量的初始化在什么时候

  

### 虚函数

[虚函数实现机制详解](https://www.jb51.net/article/189240.htm)

#### 动态绑定

绑定：指函数调用与函数本身的关联，以及成员访问与成员内存地址的关联。

静态绑定：绑定过程发生在编译期间，将函数调用和响应调用所需的代码在编译期间结合起来。

动态绑定：绑定过程发生在程序运行期间，会根据对象类型来选择实际的调用函数，并把函数调用和响应调用所需的代码在运行期结合起来；

动态绑定是多态的实现机制。多态是根据对象类型来判断调用哪一个虚函数的。

#### 虚表指针什么时候产生？

每一个类维护一张虚函数表，该类的所有实例化对象共用一张虚函数表。对象在其内存空间的头部位置存放一个虚表指针，通过虚表指针找到虚函数表，进而找到将要调用的虚函数。

可见，虚表指针是每一个对象都需要有的，虚表指针是在程序对类调用构造函数进行对象实例化的时候生成的。

在实例化一个派生类对象时，需要首先调用基类的构造函数，此时this指针开头存放的是指向基类虚表的指针，再调用派生类的构造函数，此时指向派生类虚表的指针会覆盖掉原先的基类的虚表指针。

#### 虚表如何生成

一个派生类的虚表的结构直接继承自基类的虚表，但是会覆盖掉基类虚表中那些已经被派生类重写了的虚函数，并在基类虚表结构的最后添加派生类独有的虚函数。派生类和基类拥有不同的虚表，且虚表存放的地址也不一样。

#### 重载重写的区别

重载是函数名相同，函数的参数不同，根据函数的参数类型来判断应该调用哪一个函数；

重写是函数名相同，函数参数也相同，类的继承之间，派生类可以重写基类的虚函数，并根据调用函数的对象的类型来判断应该调用基类还是派生类的虚函数；

- 菱形继承下的类的大小





- 基类的析构函数不适用虚函数会发生什么

#### final和override

`final`：

- 用来修饰类，让该类不能被继承
- 用来修饰虚函数，让该虚函数不能被重写

`override`：用于避免在派生类中忘记重写虚函数的错误。`pverride`可以显式地在派生类中声明，哪些函数需要被重写，如果没被重写，编译器会报错；

#### 构造函数能不能是虚函数

不能。

#### C++构造函数和析构函数中的异常

关键字virtual。

虚函数表包含一个类的所有虚函数的地址，对虚函数的调用通过指针进行。类对象的内存空间的头部会有一个虚函数表指针，当子类对父类虚函数进行重写的时候，虚函数表会更新相应虚函数地址。

不同的类对象共用一张虚函数表。

通过引用或指针调用虚函数时，在运行时确定。通过值调用的虚函数在编译时确定。

C语言构造虚函数：设置父类子类关系，设置虚函数表，设置虚表指针并指向虚函数表，填充虚函数表，更新重写的虚函数指针

纯虚函数是为了规范接口，规定继承这个类的程序员必须实现这个函数。

构造函数和析构函数调用虚函数不好是因为这两个函数的执行依赖父类子类构建和析构的先后顺序。

### 析构函数与虚函数

## 设计模式

- 在游戏中有哪些应用场景

### 单例模式

保证该类只有一个实例，且提供该类的一个全局访问点。分为饿汉式单例和懒汉式单例。

```c++
// 懒汉式单例，用到时再创建实例
class A{
private:
	static  volatile A instance = null;	// 保证线程同步
	A(){}	// private避免类在外部被实例化
public:
    static A getInstance(){
    	if(intance == null){
            instance = new A();
        }
        return instance;
    }
};
```

```c++
// 饿汉式单例，类一旦加载就创建一个实例，保证在调用getInstance前一定有实例
class A{
private:
    static final A instance = new A();
    A(){}
public:
    static A getInstance(){
        return this-instance;
    }
};
```

### 观察者模式

又叫发布-通知模式，红绿灯模式。通知类会发布通知，接收通知的类对象作出反应。

需要抽象发布类和具体发布类，以及抽象观察者类和具体观察者类。

- 抽象发布类中有观察者列表成员变量和通知发布成员函数，观察者管理成员函数，即能增加或删除观察者到观察者列表；
- 具体发布类是抽象发布类的派生类，负责实现通知发布函数；

- 抽象观察者类是一个接口，拥有response成员函数；

- 具体观察者类中有反应函数，根据通知采取措施。

```c++
class Subject{	// 抽象发布者类
public:
    void Attach(Observer *pObserver){
        this->m_ObserverList.push_back(pObserver);
    }
    void Detach(Observer *){
        this->m_ObserverList.remove(pObserver);
    }
    virtual void Notify() = 0;
protected:
    std::list<Obeserver *> m_ObserverList;	// 观察者列表
};

class ConcreteSubject{
public:
    void Notify();
}
void ConcreteSubject::Notify(){
    std::list<Observer *>::iterator it = Subject::m_ObserverList.begin();
    while(it != Subject::m_ObserverList.end()){
        (*it)->Upadte(1);
        ++it;
    }
}

class Observer{	// 抽象观察者类
public:
    virtual void Update(int) = 0;
};


class ConcreteObserver : public Observer{
public:
    ConcreteObserver(Subject *pSubject) : m_pSubject(pSubject){}
    void Update(int value){
        cout << "get!" << value << endl;
    }
private:
    Subject *m_pSubject;
};
```



## 其他

- 匿名函数 lamada

  匿名函数使用场景：不想给函数取额外命名，想内嵌到代码处方便可读性，同时不想开一些全局变量来存储函数变量。

  [别人的笔记](https://blog.csdn.net/lyn631579741/article/details/108763558)  [leetcode实践](https://leetcode.cn/problems/regular-expression-matching/solution/zheng-ze-biao-da-shi-pi-pei-by-leetcode-solution/#comment)

  ```c++
  // =[&](int x) 默认以值捕获所有变量，但是x例外，x通过引用捕获
  // 从c++14开始，lamada表达式开始支持泛型，其参数可以使用自动推断来获取类型，而不需要显示声明
  auto evt_set_status_x = [&](EventType x)
  {
    status[x] = true;/*通过引用捕获的变量 我们可以进行修改变量的数据*/
  };
  ```

  关于匿名函数的返回类型，如果lamada函数体内存在多个`return`语句时，编译器就不能推断其返回类型，就要在前面加上`auto`来表示返回类型。

  ```c++
  // 通过引用捕获变量i j
  // 需要返回类型
  auto matches = [&](int i, int j) {
      if (i == 0) {
          return false;
      }
      if (p[j - 1] == '.') {
          return true;
      }
      return s[i - 1] == p[j - 1];
  };
  ```

  

### static_cast

用法：

- 基本的数据类型转换，及指针之间的转换：

  ```c++
  char a = 'c';
  int b = static_cast<int>(a);
  
  char* pa = &a;
  void* pb = static_cast<void*>(pa);
  
  int c = 2;
  double d;
  d = static_cast<double>(a/c);	// 错误！！！
  ```

- 基类与子类成员函数指针的转换

  ```c++
  class A{
  public:
      void set(){}
  };
  class B:public A{
  public:
      void set(){}
  };
  typedef void(A::*PS_MFunc)();	// 指向基类A的成员函数指针
  PS_MFunc func = &A::set;
  func = static_cast<PS_MFunc>(&B::set);	// 基类指向子类成员函数指针，必须进行转换
  ```

- 基类与子类指针或引用之间的转换

  子类指针或引用转换为基类表示--安全；

  基类指针或引用转换成子类表示--危险，没有动态类型检查；

  ```c++
  class A{};
  class B:public A{};
  class C:public A{};
  class D{};
  A objA;
  B objB;
  A* pobjA = new A();
  B* pobjA = new B();
  C* pobjA = new C();
  D* pobjA = new D();
  
  objA = static_cast<A&>(objB);	// 派生类B引用 转换为基类引用
  objA = static_cast<A>(objB);
  objB = static_cast<B>(objA);	// 错误
  
  pobjA = pobjB;	// 基类指针指向子类对象
  pobjA = static_cast<A*>(pobjB);	// 子类指针转换为基类表示
  pobjC = static_cast<C*>(pobjB);	// 错误，派生类指针之间不能转换
  ```


### dynamic_cast

主要用于有继承关系的多态类（[基类](https://so.csdn.net/so/search?q=基类&spm=1001.2101.3001.7020)必须有虚函数）的指针或引用之间的转换。

1.通过dynamic_cast，将派生类指针转换为基类指针（上行转换），这个操作与static_cast的效果是一样的。

2.通过dynamic_cast，将基类指针转换为派生类指针（下行转换），dynamic_cast具有类型检查的功能，比static_cast更安全（如果转换的是指针，失败时会返回空指针；如果转换的是引用，会抛出std::bad_cast异常）

# 公司实战面经

## 腾讯天美

- 堆和栈分别是什么？怎么维护的？堆和栈能否相互转换，怎么转换？

堆和栈之间并没有一个固定的界限。

在程序编译链接生成可执行程序之后，堆和栈的起始地址就已经固定了，但是堆是向高地址增长，栈是向低地址增长，堆和栈共用全部的自由空间，只不过各自的起始地址和增长方向不同。

如果在堆和栈增长到相互覆盖时，会产生**“堆栈冲突”**，这种错误只能靠设计者自己解决。

- vector的扩容机制，会重新开辟空间吗？

vector的内存空间只会增长，不会减少。vector内的元素连续存放，vector有一个capacity()函数用来获取当前的vector的剩余预留空间。每当vector需要扩容时，都会对当前容量加倍扩容并重新分配vector的内存空间。

vector的删除操作并不会使vector的占用空间发生改变，即使clear()也不会。如果需要动态缩小空间，建议使用另一个容器 deque。

- 日常使用中，迭代器失效的情况？

- 怎么对vector某个index的元素高效删除？
- 智能指针的use_count()存放在哪

## 字节

- 在main函数之前执行一段代码

- inline函数的作用

编译器会将inline函数直接嵌入到调用语句处，而不是生成函数调用语句。

优点是：执行速度快

缺点是：复制代码段，浪费内存空间

类内成员函数隐式为内联函数，建议将逻辑简单的函数定义在类内，而涉及到递归循环的函数定义在类外的普通函数，如果内联函数中有循环递归，则会不停地复制代码段。

- 指针和引用的区别

C++primer中的对象是指 一块能存储数据并拥有某种类型的内存空间，一个对象有值和地址，运行程序时，计算机会为该对象分配空间存储值，并可以通过内存地址访问到值。

指针本质上依然是对象，只不过指针的值是其他对象的地址，通过指针就可以访问到其他对象。

而引用是变量的别名，定义一个引用的时候，程序把该引用和它的初始值绑定在一起，而不是另外拷贝一次初始值。**引用必须在声明的时候就初始化。**这也是类内初始化成员列表的存在意义之一。**引用一经声明就不可以绑定到其他对象。**相对于指针，引用由编译器保证初始化，一定不为空，因此不需要检查是否为空，提高了效率。

更深层次的讨论，引用和指针的汇编代码一模一样，引用其实就是C++的语法糖，可以看作编译器自动完成取地址、解引用的常量指针。

- 函数传参有哪几种？

- static_cast能否将一个指针类型转换为另一种？

- 析构函数为什么建议设为虚函数？
- 在C++中声明一个空类会发生什么？

## 快手

- 设计模式用过哪些？

单例模式：一个类只有一个它的实例，且提供唯一一个全局访问点，有两种实现方式，懒汉式单例和饿汉式单例。分别是 一种是new一个static类的实例，getInstance只是返回this->instance；另一种是在getInstance函数中判断该类实例是否存在，若不存在则new，存在则返回this->instance。

```c++
public class LazySingleton {
    private static volatile LazySingleton instance = null;    //保证 instance 在所有线程中同步

    private LazySingleton() {
    }    //private 避免类在外部被实例化

    public static synchronized LazySingleton getInstance() {
        //getInstance 方法前加同步
        if (instance == null) {
            instance = new LazySingleton();
        }
        return instance;
    }
}

public class HungrySingleton {
    private static final HungrySingleton instance = new HungrySingleton();

    private HungrySingleton() {
    }

    public static HungrySingleton getInstance() {
        return instance;
    }
}
```



观察者模式：一个类的消息或反馈会影响其他的类。需要设计一个抽象目标类，具体目标类，和多个具体观察者。其中抽象目标类能够发布通知，并增加或删除观察者列表；具体目标类负责实现具体的通知操作。多个具体观察者中要有response()函数

## 