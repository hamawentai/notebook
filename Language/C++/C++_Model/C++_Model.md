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

