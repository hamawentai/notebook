# 对象

C语言本身没有支持“数据和函数”之间的关联性，这种程序方法是程序性的。

结构（struct/class）变为C++后增加的成本：

* member functions在class内的声明，但不会出现在object中，**每一个non-inline member function**只会诞生一个函数实体。
* 拥有多个inline function则会在每一个是用处产生一个函数实体。（在使用类内联函数时，会在调用处展开）

以上特性并未带来任何空间或执行期的成本。

**C++的布局以及存取时间上主要是virtual引起的：**

* virtual function
* virtual base class：多继承

## 对象模式

两种class data members：

* static
* nonstatic

三种class function members：

* static
* nonstatic
* virtual

### 简单对象模型

<img src="pic/1.png" style="zoom:15%;" />

每个slot代表一个指向data member或是function member的指针。这样避免了member有不同的类型，导致需要不同的存储空间的问题。

但这个模型并未应用（why）。

### 表格驱动对象模型

<img src="pic/2.png" style="zoom:15%;" />

将data与function抽离出来，data member table存有实际的数据，而function member table则存有指向实际function的指针。

但这个模型也未被使用，而是成为了virtual function的一部分。

### C++对象模型

<img src="pic/3.png" style="zoom:50%;" />

* non-static data member放在每个对象内
* static data member则放在类对象之外
* static/non-static function member则放在所有类对象之外
* virtual function
  * 每一个类拥有一个存有该类所有虚函数指针的表格（virtual table=vtbl）
  * 每个类对象有一个指针（virtual pointer=vptr），指向所属类的vtbl。vptr的设置与重置由构造函数、析构函数与拷贝运算符完成。每个类的type_info object（支持RTTI flag），又在vtbl中指出，通常放在第一个slot处。

优点：时间和空间上的高效（存取次数）

缺点：一旦non- statc data member发送改变则需要重新编译（而上面的双表格模型则提供了一层间接性不需要重新编译）

### 加上继承后的模型

早期的iostream的实现：

<img src="pic/4.png" style="zoom:10%;" />

在虚拟继承下，base class（即此处的ios）被派生多次也永远只有一个实体（subobject）。

简单对象模型中，每一个基类都被派生类的一个slot指出。（存在间接性存取，类对象不会因为基类大小的改变而改变。）

或者说抽象出一层base table（包含一个类的所有基类，类似于虚表）：

<img src="pic/1.jpg" style="zoom:50%;" />

这两种方法都会随着继承深度的增加导致间接的次数增加。

C++最初的继承模型不使用任何间接性：基类的数据成员被直接放在派生类中，这样提高了访问效率，但是导致：基类的任何改变都将导致派生类重新编译。

引入虚基类后，会在类对象中为每一个有关联的虚基类加上一个指针。

### 对象模型对程序的影响

不同的对象模型将导致“现有程序必须修改”，以及“必须加入新的代码”。

example：

```cc
X foobar() {
    X xx;
    X *px = new X;
    xx.foo();
    px->foo();
    delete px;
    return xx;
}

// 扩展后
void foobar(X &r) {
    // 构造r
    r.X::X();
    px = _new(sizeof(X));
    if(px != 0) {
        px->X::X();
    }
    // 不需使用virtual机制
    foo(&r);
    // 使用virtual机制，从虚表中找到实际用到的函数
    (*px->vtbl[2])(px);
    // 扩展delete px
    if(px != 0) {
        // 实际调用析构函数
        (*px->vtbl[1])(px);
        _delete(px);
    }
    return;
}
```

<img src="pic/5.png" style="zoom:10%;" />

* vtbl[0]：X的type_info object
* vtbl[1]：X::~X()
* vtbl[2]：X::foo()

## 关键字的差异

struct只是方便C-->C++。

C++中，**凡是处于同一个access section的数据，必定保证以其声明次序出现在内存布局中。**然而**被放在不同access section中的数据排列次序就没有这个规定。**同理，**基类与派生类的数据成员也没有强制的先后顺序。**

C struct在C++中的唯一用途是：需要传递一个class对象的全部或部分到某个C函数中时，struct声明可以讲数据封装，并保证拥有与C兼容的空间布局。

## 对象的差异

三种程序设计模型：

* 过程模型
* ADT：struct+funtion
* OOP：类+方法

C++支持多态的方法：

* 将派生类指针转换为基类指针

  ```cc
  Base *bp = new Derived();
  ```

* 虚函数机制

* `dynamic_cast`与`typeid`

  ```cc
  Derived *dp = dynamic_cast<Derived*>(bp);
  ```

多态时经由一个共同的接口来影响封装，这个接口以virtual funtion机制触发，在执行期根据object的真正类型解析出到底那个函数实体被调用。

一个类对象需要的内存大小：

* 非静态数据成员的总和
* 由于对齐而padding的空间
* 为了支持virtual而产生的负担

指针和引用的大小都是固定不变的，引用通常是以一个指针来实现的。

## 指针的类型

从内存的角度来说一个ClassA\*a与ClassB\*b没有什么不同，它们之间的差异只存在其寻址出来的object类型不同（表示法、内容（即地址类型）都是一样的）。“指针类型”会告诉编译器如何解释某个特定地址中内存内容及其大小。

<img src="pic/6.png" style="zoom:10%;" />

一个ZooAnimal类型的指针，会将其代表的1000地址开始解释为一个ZooAnimal的对象布局。

而如果将其变为一个void\*，编译器将不知道1000地址开始会涵盖一个怎样的空间。**所以这就是一个类型void\*只能喊有一个地址，而不能通过它操作所指的object的缘故。**

所以，cast只是一种编译器指令，大部分情况下并不改变一个指针所指的真正地址，只影响“被指出的内存大小及其内容”的解释方式。

### 加上多态

<img src="pic/7.png" style="zoom:10%;" />

派生类会加上基类的那一部分：

<img src="pic/8.png" style="zoom:15%;" />

```cc
Bear b;
ZooAnimal *pz = &b;
Bear *pb = &b;
```

pz与pb都指向b的第一个byte（即地址1000）：

* pb涵盖整个b（1000～1023）
* pz只涵盖b中的ZooAnimal部分（1000～1015）

pz不能直接操作b中属于Bear的部分，必须通过virtual机制：

```cc
pz->cell_block;//不行
((Bear*)pz) -> cell_block;
// or
Bear *b2 = dynamic_cast<Bear*>(pz); //成本较高，属于运行期行为
```

类型信息的封装并不是维护在指针中，而是维护在link中。（此link存在于“object的vptr和vptr所指的virtual table”。）

```cc
Bear b;
ZooAnimal za = b; // 会引起切割
za.rotate(); // 调用的是ZooAnimal::rotate（）
```

* rotate为什么调用的是ZooAnimal而不是Bear的实体：

  非指针/引用不会发生多态，指针/引用支持多态的原因：

  它们并不会引发内存中“与类型有关的内存委托操作”，会受到改变的只是它们所指的内存的“大小内容和解释方式而已”

* 将b拷贝给za时，为什么za的vptr没有指向（被拷贝）Bear的virtual table：

  每个vptr对应自己虚函数的那部分，基类部分改变不影响派生类部分，基类指针接管基类部分，派生类接管派生类部分

# 构造函数

## 默认构造函数

在需要的时候才会被构造出来：

* 程序需要
* 编译器需要

```cc
class Foo {public: inv val; Foo *next};
void foo_bar() {
  Foo bar;
  // ...
}
```

上述代码不会产生默认构造函数。如果程序需要默认构造函数，那么就需要自己合成。

即使有需要为Foo合成默认构造函数，这个默认构造函数也不会将Foo的数据成员初始化为0。

### 带有默认构造函数的成员类对象

一个类没有任何构造函数，而它有一个成员是一个对象，这个成员有默认构造函数，那么这个类的隐含的默认构造函数就是“non-trivial”（有效的），编译器就会为该类生成一个默认构造函数。（不过这个合成操作只会在需要使用构造函数时才会被调用。）

```cc
class A {
 public:
  A(){}
};
// 因为A有构造函数，而B没有但却有一个成员是A，所以编译器会为B生成一个默认构造函数
// 但是该构造函数不会初始化i
class B {
 public:
  int i;
  A a;
};
// B中被合成默认构造函数类似于
inline B::B() {
  // 伪代码
  foo.Foo::Foo();
}
```

被合成的默认构造函数只满足编译器的需要，而不是程序的需要。所以，B中i不会被初始化。

```cc
class B {
 public:
  	int i;
  	A a
    B():i(0) {}
};
```

此时由于B已经存在了一个默认构造函数，那么编译器为了初始化a，不会再为B生成默认构造函数，而是扩展它：

```cc
// 扩展B的默认构造函数 伪码
B::B():i(0) {
  foo.Foo::Foo();
}
```

如果存在多个成员对象，C++则会以成员对象在类中的声明次序来依次调用各个构造函数。

### 带有默认构造函数的基类

一个没有任何构造函数的类派生自一个有默认构造函数的基类，那么这个派生类的默认构造函数是nontrivial。会合成一个默认构造函数，在其中调用上一层基类的默认构造函数。

如果提供了多个构造函数，但是没有默认构造函数的话，编译器会扩张每一个现有的构造函数，将一些需要调用的操作加入其中（调用基类或是成员对象的构造函数）。编译器不会再合成新的默认构造函数。

### 带有一个虚函数的类

* 类声明/继承一个虚函数
* 类派生自一个继承串，其中有一个或多个虚基类

这些也是需要合成默认构造函数的。

<img src="pic/9.png" style="zoom:20%;" />

Bell、Whistle都派生自Widget（其中含有一个虚函数）。那么会在编译期间发生扩张操作：

* 产生一个虚函数表（vtbl）
* 在每一个类对象中加入一个vptr指向vtbl

widget.flip()引发的操作将会被虚函数表重定向：

```cc
(*widget.vptr[1])(&widget);
```

同时编译器会为它们合成一个默认构造函数用以初始化每个vptr。

### 带有一个虚基类的类

<img src="pic/10.png" style="zoom:10%;" />

```cc
struct X {int i;};
struct A : virtual X {int j;};
struct B : virtual X {int k;};
struct C : A, B {int l;};
// 无法在编译期解析出pa->X::i，因为pa的真正类型会变化
void foo(const A *pa) {pa->i = 1024;};
```

cfront的做法是：在派生类对象的安插virtual base classes来完成：

<img src="pic/11.png" style="zoom:10%;" />

```cc
// foo改写伪码
// 这样不论是pa的类型A还是C都会根据vbcx指到X
void foo(const A *pa) {pa->_vbcX->i = 1024;};
```

_vbcX表示指向虚基类X的指针。

所以编译器会为没有任何构造函数的类生成默认构造函数来初始化对虚基类的操作。



所以，不在这四种情况内的而又没有声明任何构造函数的类，它们拥有的是implicit trivial的默认构造函数，实际上是不会被合成出来的。

## 拷贝构造函数

三种情况下会将一个object作为另一个object的初值：

* 赋值

  ```cc
  X xx = x;
  ```

* 传参

  ```cc
  void foo(X);
  foo(x);
  ```

* 返回值

  ```cc
  X foo2();
  ```

当一个类对象以另一个同类实体**作为初值**时，会调用拷贝构造函数（而不是赋值运算符），这会产生一个临时对象（以函数返回值作为初值时）。

```cc
C1 c_f() {
  return C1(1);
}
// C1(1) 构造
// temp = C1(1) 拷贝构造
// 离开c_f C1(1)析构
// c = temp 拷贝构造
// 析构temp
// mian结束析构c
void t17() {
  C1 c = c_f();
}
// C1(1) 构造
// temp = C1(1) 拷贝构造
// 离开c_f C1(1)析构
// c持有 temp 的引用
// mian结束析构temp
void t18() {
  const C1& c = c_f();
}
C1 getC1_2(int i) {
  return {i};
}
// 效果等同于 t18 flag
```

### 默认深拷贝初始化

如果类没有提供显式的拷贝构造函数，编译器会以深拷贝的方式完成拷贝，即**不会拷贝其中的成员类对象，而是以递归的方式调用其（成员类对象）。**

```cc
struct A {
  int a,b;
};
struct B{
  int i;
  A a;
};
```

B先会拷贝i，然后调用A的拷贝构造函数来拷贝a。

“如果一个类为定义拷贝构造函数，编译器就会为它产生一个。”这是不对的，实际上是：**默认构造函数与拷贝构造函数只在必要时（类需要递归拷贝，即在需要深拷贝时）才由编译器产生。**

```cc
A a;
void f() {
  A b= a;
}
// 1. 不会合成拷贝构造（浅拷贝）
class A {
  public:
  	int a,b,c;
};
// 2. 需要合成拷贝构造（深拷贝）
class A {
  public:
  	int a,b,c;
  	B cb;
};
// 此时A需要合成一个拷贝构造函数，不仅拷贝A自身的a,b,c还会调用B的拷贝构造函数去拷贝B的成员
class B {
  int a,b,c;
  B(const B&);
};
```

### 不展现浅拷贝

* 基类/成员对象中显式定义了拷贝构造函数

  编译器必须将基类/成员对象的构造函数的调用操作安插到自己的合成的拷贝构造函数中

* 类声明了一个/多个虚函数

* 类派生的链中存在虚基类

#### 重新设定虚表指针

编译期间的两个扩张操作：

* 增加一个虚函数表（vtbl），保存每个虚函数的地址
* 为类对象产生一个指向vtbl的虚函数表指针（vptr）

<img src="pic/12.png" style="zoom:12%;" />

所以必须正确设定vptr，**当编译器导入一个vptr到类中时，该类就不会再展现浅拷贝。**相应的，会合成一个拷贝构造函数，将vptr正确的初始化。

example：

<img src="pic/13.png" style="zoom:10%;" />

```cc
Bear yogi;
Bear winnie = yogi;
ZooAnimal franny = yogi;
```

将`yogi`的vptr拷贝给`winner`是安全的，但将之拷贝给`franny`是非法的。

<img src="pic/14.png" style="zoom:15%;" />

需要将`franny`的vptr指向ZooAnimal：

<img src="pic/15.png" style="zoom:15%;" />

也就是说，**合成出来的ZooAnimal拷贝构造函数，会明确设定对象的vptr指向ZooAnimal的虚表。**

#### 处理虚基类

除了上述的第三条，还有**如果一个类对象以另一个对象作为初值，而后者有一个虚基类子对象**，那么也会使浅拷贝失效。

<img src="pic/16.png" style="zoom:20%;" />

编译器承诺：会让**派生类中的虚基类的对象（即Raccoon中的ZooAnimal部分）的位置准备好。**编译器会产生代码，**放在Raccoon每个构造函数的头部，来调用ZooAnimal的默认构造函数、将vptr初始化，并定位出Raccoon的ZooAnimal部分。**

* 当一个类对象以另一个同类的类对象作为初值时，浅拷贝不会失效（如Raccoon之间的拷贝）。

* 当一个类对象以其派生类的对象作为初值时，浅拷贝会失效（如franny=yogi）。

```cc
Raccon a;
// 简单的浅拷贝足矣
Raccon little_critter = a;

// 编译器需要明确地将little_critter的vptr初始化
RedPanda little_red;
little_critter = c;
```

<img src="pic/17.png" style="zoom:15%;" />

也有例外：

```cc
// 浅拷贝可能够用，也可能不够用
Raccon *ptr;
Raccon little_critter = *ptr;
```

因为编译器无法知道，ptr是否指向一个真正Raccon对象，或是一个派生类对象。

## 程序转换语意

```cc
X foo() {
  X xx;
  return xx;
};
```
如下假设：
* 每次foo被调用，就会返回xx的值
* 如果foo定义了拷贝构造函数，当foo被调用时，保证该拷贝构造函数也会被调用
* 上述两点都不正确

### 明确的初始化操作

```cc
X x0;
void f() {
  X x1(x0);
  X x2 = x0;
  X x3 = X(x0);
}
```
* 重写每个定义，其中初始化操作会被剔除（C++中的定义严格来说指的是“占用内存”的行为）
* 类的拷贝构造掉用操作会被安插进去
转换后：
```cc
void f1() {
  X x1, x2, x3; // 定义被重写，初始化操作被剔除
  // 编译器安排的X的拷贝构造操作
  x1.X::X(x0); // 即 X::X(const X&xx);
  x2.X::X(x0);
  x3.X::X(x0);
}
```

### 参数初始化

把一个类对象作为参数传递给一个函数（或作为函数的返回值），相当于：
```cc
X xx;
void f(X x0);
f(xx);
```
会将实参拷贝给形参（即将xx拷贝给x）。
会要求局部对象（x0）以深拷贝的方式将实参（xx）作为初值。
一种策略是**导入暂时性的对象，并调用拷贝构造函数将它初始化，然后将暂时先对象交给函数。**
```cc
// 产生的暂时性对象
X _temp;
// 编译器对拷贝构造函数的调用
_temp.X::X(xx);
// 重写函数调用，以便使用上述暂时对象
foo(_temp);
```
有一些问题：_temp先以X的拷贝构造函数正确地设置了初值，然后再浅拷贝到x0这个局部实体中。导致foo的声明也必须发生变化：
```cc
void f(X&);
```
另一种策略是**以拷贝构建的方式，把实际参数直接建构在其应该在的位置上，该位置视函数活动范围的不同记录于程序堆栈中。在函数返回前，局部对象的析构函数（有的话）会被执行。**

### 返回值的初始化`RVI(Return Value Initialization)`

讨论的是:
```cc
X foo() {
  X xx;
  return xx;
}
```
foo()的返回值是如何从局部对象xx中拷贝出来的。
其由编译器转换的伪码如下：
```cc
void foo(X &_result) { // 加上一个额外参数
  X xx;
  // 编译器产生的默认构造函数调用操作
  xx.X::X();
  // 编译器产生的拷贝构造函数调用操作
  _result.X::X(xx);
  return;
}
// 调用foo
X x = foo();
// 被转换为
X x;
foo(x);
// ====
foo().func();
// 转化为
X x;
// ","运算符
(foo(x), x).func()
```
### 使用者层面的优化

对于foo的调用，可以改下为“计算用”的函数，提高效率：
```cc
X foo(int a) {
  // 没有了中间变量x
  return X(int a);
}
// 转换后
void foo(X& _result, int a) {
  _result.X::X(a); // 相较于之前，少了一次初始化的操作（xx.X::X();）
  return;
}
```

### 编译器层面的优化

在foo中，所有的return指令都返回有相同名字的值（named value，具名数值），那么编译器可以进行优化，**以result参数取代named return value（具名返回值）**：
```cc
// 把foo中的xx以_result替代
void foo(X &_result) {
  _result.X::X();
  // 直接使用_result,省略了一次xx到_result的拷贝操作（或xx的初始化操作）
  return;
}
```
节省了：
1)  在堆栈中预留xx的内存；
2)  调用X的默认构造函数，构造xx
3)  调用X的拷贝构造函数，构造_result
  变为了调用其构造函数
4)  调用xx的析构函数
5)  堆栈中回收xx的内存
> **需要显式的拷贝构造函数才会触发NRV的原因：**截取其中的话就是“早期的 cfront需要一个开关来决定是否应该对代码实行NRV优化，这就是是否有客户（程序员）显式提供的拷贝构造函数：如 果客户没有显示提供拷贝构造函数，那么cfront认为客户对默认的逐位拷贝语义很满意，由于逐位拷贝本身就是很高效的，没必要再对其实施NRV优化；但 如果客户显式提供了拷贝构造函数，这说明客户由于某些原因(例如需要深拷贝等)摆脱了高效的逐位拷贝语义，其拷贝动作开销将增大，所以将应对其实施NRV 优化，其结果就是去掉并不必要的拷贝函数调用。”
>

## 成员初始化列表

构造函数发生作用在于：
* 成员初始化列表
* 构造函数体中
但实际上除了某些情况下（四种情况），其实二者都是一样的：
* 初始化一个引用成员
* 初始化一个const成员（必须初始化）
* 调用用一个基类的构造函数，而它拥有一组参数
* 调用用一个成员对象的构造函数，而它拥有一组参数
```cc
class A {
  public:
    int a,b;
    String s;
    A() {
      // 构造函数体只是完成赋值操作，而初始化早已完成
      a = 1;
      b = 1;
      // 会在初始化（函数的最开始，早于程序员的代码）时创建于（初始化）一个临时的String对象（保存“hello”）
      // 再使用赋值运算符，将这个临时对象给s赋值，然后再摧毁掉这个临时对象
      s = "hello"
    }
};
// 伪码
A() {
  s.String::String(); // 初始化s
  // 产生临时对象
  String temp =  String("hello");
  // 深拷贝temp
  s.String::operator=(temp);
  // 摧毁temp
  temp.String::~String();
  a = 1;
  b = 1;
}
```
此时成员初始化列表就会发生作用：
```cc
A():a(1),b(1),s("hello") {}
// 伪码
A() {
  // 直接就初始化s了，不会产生临时对象
  s.String::String("hello");
  a = 1;
  b = 1;
}
// 实际上a,b是一个行为良好的成员
A():s("hello") {
  a = 1;
  b = 1;
}
// 与A():a(1),b(1),s("hello") {}效果一样
```
实际上，编译器会一一操作initialization list，以其在类中（public、protect、private中的相对顺序）声明的顺序在构造函数体中插入初始化操作，并**先于构造函数体中用户定义的代码**。
```cc
class C4 {
 public:
  int a, b, c;
  C4(int x) {
    // a b c啥也不会变，还是未定义的状态
    cout << a << " " << b << " " << c << endl;
    a = b = c = x;
  }
  //  warning: field 'b' is uninitialized when used here [-Wuninitialized]
  C4(int x, int y) : b(x+1), a(b), c(y) {
    // a(x), b(y), c(x)先于用户定义的代码
    cout << a << " " << b << " " << c << endl;
  }
};
```
还可以在成员初始化列表中调用类方法：
```cc
X::X(int x):a(func(x)) {}
```
也是正确的（但是func中不能依赖先于a初始化的成员变量），**这是因为此时与对象相关的this指针已经构建完成**：
```cc
// 伪码
X::X(/*this指针*/int x {
  i = this->func(x);
  a = i;
}
```
<img src="pic/18.png" style="zoom:15%;" />

# 数据

```cc
class X {};
class Y : public virtual X {};
class Z : public virtual X {};
class S : public Y, public Z {};
int main() {
  int* p;
  cout << "point size: " << sizeof(p) << endl;
  cout << sizeof(X) << endl;
  cout << sizeof(Y) << endl;
  cout << sizeof(Z) << endl;
  cout << sizeof(S) << endl;
}
// 输出
point size: 8
1
8
8
16
```
一个空的对象（如X）的大小并不是为0，**它有一个隐晦的1byte，是被编译器插入的一个char，使得这个class的两个对象在内存中有独一无二的地址。**
> C++标准中规定，“no object shall have the same address in memory as any other variable” ，就是任何不同的对象不能拥有相同的内存地址。如果是空对象就不会占内存，不占内存就地址一样了。避免指针运算为0。[参考链接](https://blog.csdn.net/weixin_42323413/article/details/84295375)
>
Y和Z大小受三个因素影响：
* 语言本身负担。
  当语言支持虚基类时，在派生类中就会存在一个指针：或指向虚基类子对象（自身从基类继承下来的哪部分）；或是一个相关表格（存放的是虚基类子对象的地址或偏移量）
* 编译器对特殊情况的优化处理。
  X中的1byte的影响也存在于Y和Z中。
  或者另一种策略：空虚基类对象直接成为派生类的一部份
  *mac上输出为什么没体现出虚指针or对齐？ flag*
* 对齐。
  X为1byte，Y、Z为了对齐
exmaple（书上例子）：

<img src="pic/19.png" style="zoom:15%;" />

一个虚基类子对象只会在派生类中存在一份实例，**不管它在class继承体系中出现了多少次。**A的大小取决于：
* 共同实体的大小（即X）为1
* Z/Y的大小减去已经被计算的X的大小 8+8-2=14
* A自己的大小  0
* 对齐。14 + 1 + 1 = 16
> 各种编译器的实现还有优化不同，书上仅供参考，关键是那1byte。
>
C++保持对C struct的兼容。
* 它把数据以及继承来的非静态数据成员直接存放至每一个类对象中
* static数据成员则放在一个global数据段(segment)中，不会影响个别类对象的大小
* static数据成员永远存在且只存在一份实体（即使该类还没有对象实体，但是template的static稍有不同）

## 数据成员的布局

```cc
class A {
  public:
    int a,b,c;
    float d,e,f;
};
```
C++标准中，同一个access section（即public、protect、private）中的成员排列符合：较晚出现的成员在类对象中有着较高的地址。

<img src="pic/20.png" style="zoom:15%;" />

至于编译器合成的内部使用的数据成员（用于支持对象模型，如vptr）则通常被放在所有明确声明的成员的最后/前端。而多个access section之间可以自由排列。（section不会增加内存负担，在一个section声明八个成员与在八个section在分别声明一个成员的大小是一样的。）

## 数据成员的存取

### 静态成员

静态成员视为一个全局变量（只在class的生命范围内可见）。对一个静态成员取地址，会得到一个**指向数据类型的指针**，而不是**指向类数据成员的指针**。
对于不同类中的相同名字的静态对象，编译器会对它们取一个不同的名字（可以倒推回去）。

### 非静态成员

非静态成员存在每个类对象中，无法直接存取，只能通过`implict class object`：
```cc
int A::f(int a) {
  return i+a;
}
// 伪码
int A::f(A* const this, int a) {
  return this->i + a;
}
```
编译器会根据对象的地址（this指针）加上数据成员的偏移量进行存取：
```cc
origin.y = 0;
// 伪码
&origin + (&A::y - 1);
```
指向数据成员的指针，其偏移量总是被加上1（所以需要-1），这样使得编译系统可以区分“第一个指向数据成员的指针，用来指出类的第一个成员” 和 “一个指向数据成员的指针，没有指出任何成员”这种区别类似数据首元素的地址和数组的地址。flag
每一个非静态成员对象（以及基类子对象）的偏移量在编译期可知。
虚拟继承将为经由基类子对象存取的数据成员加上一层间接性：
```cc
A *p;
p->x = 0;
```
x若是一个结构体、类对象、单一继承、多重继承它们存取情况都一样。**若是一个虚基类的成员，存取速度就会慢一点。**
因为：p是一个指针，那么在编译期无法确定它真正指向的类型，只能延迟到延迟期才能确定，所以需要引入一层间接导引。但是若是A a的话，a的类型在编译期就明确确定了其类型，也就确定了x的偏移量。
所以，以下代码是有存取差距的：
```cc
A *p, a;
p->x;
a.x;
```

### 继承与数据成员

一个派生类 = 自己的成员 + 它派生的基类的成员。
自己的成员与继承的成员的顺次没有强制规定。

#### 只要继承不要多态

一般而言**非虚继承并不会增加空间或存取上的额外负担。**

<img src="pic/21.png" style="zoom:15%;" />

单一继承且没有虚函数时的内存布局如上。

由于C++保证“出现在派生类中的基类子对象”的完整性以及对齐操作，将会导致类的大小变化：
[参考链接1](https://zhuanlan.zhihu.com/p/30007037)
[参考链接2](https://www.cnblogs.com/zrtqsk/p/4371773.html)
>
为什么需要内存对齐？
1.  平台原因（移植原因）：不是所有的硬件平台都能访问任意地址上的任意数据的；某些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。
2.  性能原因：数据结构（尤其是栈）应该尽可能地在自然边界上对齐。原因在于，为了访问未对齐的内存，处理器需要作两次内存访问；而对齐的内存访问仅需要一次访问。
>
flag
<img src="pic/22.png" style="zoom:25%;" />
***
*书上版本*
<img src="pic/23.png" style="zoom:15%;" />
