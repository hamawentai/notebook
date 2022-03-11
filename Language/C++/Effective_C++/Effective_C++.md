# 视C++为语言集合

将C++视为多种相关语言的集合，以下是几种子语言（sublanguage）：

* C：C++仍是以C为基础
* Object—Oriented C++：封装、继承、多态、虚函数
* Template C++：泛型编程
* STL：一个模板库，**迭代器和函数对象都是在C指针之上塑造而成**

# 以`const`、`enum`、`inline`替换`#define`

以编译器替换预处理器。

```cpp
#define CAT 12
```

使用的CAT不会进入符号表，在代码中就是一个12，所以出问题时不会说是CAT。

常常使用常量`const`来替换`#define`：

* 指针常量定义式通常放在头文件中，所以有必要将指针（而不是指针所指之物）声明为const
* class的专属常量
  * 为了将常量限制于class中，就得让它成为class的一个成员
  * 为了确保常量至多只有一份实体，就得让它成为static


```cc
class A {
 private:
  static const int n = 5; // 常量声明式
  static int m; // 只能声明，不可以赋值，在类外定义
};
```

n是声明式而非定义式，它既是class的常量又是static且还是整数类型（如int、char、bool），则只要不取它们的地址就可以声明并使用它们而无需提供定义式。

也可以在**实现文件（而非头文件）**中加上定义式（这样就能取地址了）：

```cc
const int A::n; // 必须是全局的而不是局部的
```

无需赋值的原因：由于class常量在声明获得了初值，所以定义时就不可再设处置。

无法用#define创建一个class的专属常量，**因为#define**不重视作用域，一旦宏被定义，它就在其后编译过程中有效（除非被#undef）。

一个枚举类型的数值可当作ints被使用（即作为常量来声明数组的长度）。enum类似#define而不像const，**取const地址是合法的，而取一个enum（or #define）的地址就是非法的。**

```cc
class B {
public:
    // 是模版元编程的基础技术 flag
    enum {
        n = 5 // 5就成为了一个记号
    };
    int s[n];
};
```

如果不想让他人获得一个pointer或reference指向某个整数常量，就可以使用enum。优秀的编译器不会为“整数型const对象”设定另外的存储空间（除非创建了一个指针/引用指向了该对象）（直接硬编码？），而**enum和#define绝不会导致非必要内存分配**。

#define常被用来实现宏(macros)，宏像函数，但不会带来函数调用带来的而外开销（类似将函数在调用处展开）

```cc
// 用a，b中的较大值来调用f
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
int a=5, b=0;
// f((++a) > (b) ? (++a) : b)
CALL_WITH_MAX(++a, b);  // 若是++a比b大会导致两次++a
```

为了避免这种情况可以使用inline函数替代之，若还需要泛化性，就还可以使用template inline。

SO：

* 对于单纯的常量，最好使用const或enum代替#define
* 对于形似函数的宏，最好使用inline函数代替#define

# 使用const

令函数返回一个常量值，可降低错误。

```cc
const A operator*(const A&l, const A &r);
// 返回值无const
(a * b) = c; // 在a、b的结果上调用operator=
```

必须避免无端地与内置类型不兼容。

## const成员函数

* 使得class接口易于理解，明白那个函数可以改动对象
* 使得操作const对象成了可能

**两个成员函数如果只是常量性不同，就可以被重载。**

```cc
class A {
  public:
  	const char& operator[](size_t pos) const {return text[pos];}
  	char& operator[](size_t pos) {return text[pos];}
 	private:
  	string text;
};
```

只要重载operator[]并对不同版本给予不同的返回类型，就可以令const和non-const获得不同的处理：

```cc
tb[0]; // 读non-const
tb[0] = 'x'; // 写non-const
ctb[0]; // 读const
ctb[0] = 'x'; // 错误：试图写const
```

如果一个对象中一部分不想被更改，而另一部分即使在const函数中也要被更改，则需要使用mutable：

```cc
class A {
  private:
  	mutable bool flag;
};
```

## 避免const和non-const成员函数的重复

比如重载operator[]，会有const和non-const两个版本，基本逻辑是一致，为了避免重复，就需要一个调用另一个。

* non-const调用const

  ```cc
  char& operator[](size_t pos) {
    // step1. 将this变为const从而调用const版本(必要的，不然会不断调用自己即non-const版本)
    // step2. 将返回的const转为非const
    return const_cast<char&>(static_cast<const A&)(*this)[pos];
  }
  ```

* const调用non-const

  const成员函数承诺不改变对象的状态，但是在const中调用non-const则破坏了这一承诺。

# 确定对象在使用前已被初始化

```cc
int x; // x被初始化为0
class A {
  public:
  	int x; // 不一定会被初始化为0
};
```

C part of C++且初始化会带来运行期成本，那么就不保证发生初始化。

non-C part of C++上述规则就会发生变化，这就是为什么：为什么array（来自C part of C++）不保证内容被初始化，而vector（来自 STL part of C++）却有此保证。

**确保每个构造函数都将对象的每一个成员初始化。**

```cc
class A {
  public:
  	int a,b,c;
  	A(int aa, int bb, int cc) { 
      a = aa; // 属于赋值而非初始化
      b = bb;
      c = bb;
    }
  	A(int a, int b, int c):a(a),b(b),c(c) {} // 初始化，此时构造函数本体不再需要任何动作
};
```

第二个实现效率高于第一个。

第一个实现步骤：

* 调用default构造函数为a、b、c初始化
* 使用构造函数体对其赋值

特别是对于const和引用是必须要初值的。

class成员变量总是以其声明次序被初始化。

non-local static对象的初始化次序：

* static对象寿命直到程序结束为止
* stack和heap-based都不是static对象
* global、namespace、classe、函数、file作用域中被声明的是local static对象
* 其他对象就都属于non-local static对象，**它们的析构函数知道main结束才会被调用**

一个non-static函数/成员需要用到non-local static成员，就必须保证non-local static先于non-static成员/函数初始化，但这是无法保证的。

需要使用单例模式将non-local static变为local static。

```cc
// before
static A a;
// after
A& getA() {
  static A a;
  return a;
}
```

# C++编写并调用的函数

空类，编译器会生成：拷贝构造函数、拷贝赋值运算符、析构函数，它们都是public且inline的。

*与C++ Model相呼应。*

* C++并不允许让引用改指向不同对象，如果想对一个内含引用成员数据的类中支持拷贝运算，则需要自定义其拷贝赋值运算符
* 如果基类的拷贝运算符为private，那么编译器将不会为其派生类生成拷贝赋值运算符

# 不想编译器自动生成函数就该拒绝

* 将拷贝构造函数or拷贝赋值运算符声明为private

  不安全，因为其成员与友元还是能调用。（可以只声明而不定义）。

C++11好像有其他办法，忘了。

# 多态基类声明virtual析构函数

只有当类中含有至少一个virtual函数才为它声明virtual 函数。

但这也不是万分安全的：

如：一个类继承了一个基类，而该基类的析构函数是non-virtual的，然后在delete 基类指向该类的指针时将不会释放该类的资源。

# 异常不能逃离析构函数

C++并不禁止析构函数抛出异常，**但不鼓励这么做。**

pass

# 不在析构函数和构造函数中调用virtual函数

这样是不会生效的，在析构/构造中调用虚函数时，该函数实际上隶属于正在执行析构/构造类的函数。

* 由于析构的顺序是先派生类再基类：此时派生了都没了，就不能将虚函数下降至派生类。
* 由于构造函数的顺序是先基类再派生类：此时派生类还没生成，也不能将虚函数下降至派生类。

# 令operator=返回一个引用 *this

赋值可以连锁赋值：

```cc
x = y = z = 15;
// 等价
x = (y=(z=15));
```

**为了实现连锁赋值，`=`运算符必须返回一个引用指向操作符左侧的实参。**

# 在operator=中处理自赋值

自赋值发生在对象赋值给自己时。

```cc
A& A::operator=(const A& r) {
  delete p; 
  p = new P(r.p);// 自赋值情况下p早已被删除
  return *this;
}
```

修正：证同测试：

```cc
A& A::operator(const A&r) {
  if(this == &r) return *this;
  // ...
  p = new P(r.p); //如果new发生异常也会出现错误，不过这种几率很小
  // ...
}
```

更好的版本：

```cc
A& A::operator=(const A &r) {
  P *np = p;
  p = new P(r.p); // 此时将delete放到了后面，就不会出现自赋值错误
  delete p;
  return *this;
}
```

# 复制对象时勿忘每个成分

当自定义拷贝函数时：当自定义实现代码几乎必然出错时，编译器不会爆出。

如：开始为A的每个成员逐一赋值时，是正确的，但是当A新增成员时，编译器却不会告诉你需要修改自定义的拷贝函数。

其次，自定义版本还需要**包括从基类继承的成员，包括private**，就需要调用对应的基类的函数。

不要让拷贝构造函数与拷贝赋值运算符互相实现（一个调用另一个），要不就是此时对象还未初始化（赋值调用拷贝），要不就是不能成功。

# 以对象管理资源

```cc
void f() {
  A *a = new A();
  // do something seg1
  delete a;
}
```

若是seg1处提前返回，或者发生异常，将会导致a的资源不会被释放。

为了确保资源总是被释放，需要将资源放入对象中，当资源离开作用域时，对象的析构函数会自动释放哪些资源。**把资源放进对象便可以依赖C++的“析构函数调用机制”来确保资源被释放。**

* 获得资源后立即放入管理对象内
* 管理对象使用析构函数确保资源被释放

