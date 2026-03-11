# enable_shared_from_this

错误代码，引发双重删除：

```c++
// 1. 外部创建shared_ptr
auto session = std::make_shared<TcpSession>(fd);
// 此时：控制块1的引用计数=1，管理着TcpSession对象

// 2. 调用handle_read方法
session->handle_read(); 
// 在handle_read内部：
void handle_read() {
    // 3. 错误地创建新的shared_ptr
    auto self = std::shared_ptr<TcpSession>(this);
    // 此时：控制块2的引用计数=1，也管理着同一个TcpSession对象
    
    // 4. 启动异步操作（假设异步操作持有self）
    async_operation([self]() { 
        // 异步操作期间，控制块2的引用计数保持为1
    });
}
// 5. handle_read函数结束，self局部变量销毁
// 控制块2的引用计数减为0，但异步操作中的lambda还持有self的副本
// 所以控制块2的引用计数仍为1，对象不会被立即删除

// 6. 外部作用域结束，session销毁
// 控制块1的引用计数减为0 → 删除TcpSession对象！

// 7. 异步操作完成，lambda中的self副本销毁  
// 控制块2的引用计数减为0 → 再次删除同一个TcpSession对象！（双重删除）
```

修复方式一：

```c++
class TcpSession {
public:
    void handle_read(std::shared_ptr<TcpSession> self) {
        // 通过参数传递，确保使用同一个shared_ptr
        async_write([self](const std::error_code& ec) {
            self->on_write_complete();
        });
    }
};

// 使用
auto session = std::make_shared<TcpSession>();
session->handle_read(session); // 显式传递
```



enable_shared_from_this实现：

```c++
// =========== enable_shared_from_this =========
template<typename T>
class enable_shared_from_this {
private:
    mutable weak_ptr<T> weak_this;  // 关键：存储对自身的弱引用

public:
    shared_ptr<T> shared_from_this() {
        return shared_ptr<T>(weak_this);  // 从弱指针构造shared_ptr
    }
    
    shared_ptr<const T> shared_from_this() const {
        return shared_ptr<const T>(weak_this);
    }
    
    // 内部接口，供shared_ptr构造函数调用
    template<typename U>
    void _internal_accept_owner(shared_ptr<U> const* owner, T* ptr) const {
        if (weak_this.expired()) {
            weak_this = shared_ptr<T>(*owner, ptr);  // 建立弱引用
        }
    }
};

// =========== shared_ptr =========
template<typename T>
class shared_ptr {
private:
    T* ptr;
    control_block* control;  // 引用计数控制块

public:
    // 构造函数模板
    template<typename U>
    explicit shared_ptr(U* p) : ptr(p), control(new control_block) {
        // 关键步骤：检查U是否继承enable_shared_from_this
        setup_weak_this(p);
    }

private:
    template<typename U>
    void setup_weak_this(U* p) {
        // 使用SFINAE或concept检查继承关系
        if constexpr (std::is_base_of_v<std::enable_shared_from_this<U>, U>) {
            // 调用enable_shared_from_this的内部方法
            p->_internal_accept_owner(this, p);
        }
    }
};
```





```c++

class CorrectTcpSession : public std::enable_shared_from_this<CorrectTcpSession> {
    void handle_read() {
        // 步骤3：调用shared_from_this()
        auto self = shared_from_this();
        
        // 步骤4：创建lambda并捕获self
        async_write([self](const std::error_code& ec) {
            if (!ec) {
                self->on_write_complete();
            }
        });
    } // 步骤5：handle_read结束，局部self销毁
    
    void on_write_complete() {
        // 步骤7：异步操作完成后的处理
    }
};

// 步骤1：创建对象
auto session = std::make_shared<CorrectTcpSession>();

// 步骤2：调用方法（假设在某个事件触发时）
session->handle_read();

// 步骤6：可能在其他地方session引用被释放
```



| 时间点    | 操作                                                    | 引用计数 | 控制块状态                                   | 说明                         |
| :-------- | :------------------------------------------------------ | :------- | :------------------------------------------- | :--------------------------- |
| **步骤1** | `auto session = std::make_shared<CorrectTcpSession>();` | **1**    | 新创建控制块，`weak_this`被初始化            | 对象被第一个`shared_ptr`管理 |
| **步骤2** | `session->handle_read();`进入函数                       | 1        | 不变                                         | 只是方法调用，不涉及引用计数 |
| **步骤3** | `auto self = shared_from_this();`                       | **2**    | `weak_this.lock()`成功，创建新的`shared_ptr` | 从弱指针成功创建强引用       |
| **步骤4** | Lambda捕获`self`（按值拷贝）                            | **3**    | Lambda内部的`self`副本增加引用               | 异步操作持有对象引用         |
| **步骤5** | `handle_read()`函数结束，局部`self`销毁                 | **2**    | 局部变量离开作用域                           | 但Lambda中仍有引用           |
| **步骤6** | 外部`session`被释放（如离开作用域）                     | **1**    | 外部引用消失，但异步操作仍在进行             | **对象不会被销毁！**         |
| **步骤7** | 异步操作完成，Lambda中的`self`销毁                      | **0**    | 最后一个引用消失                             | 对象被安全销毁               |

# weak_ptr

```c++
// lock() 是原子操作，整个过程不会被中断
shared_ptr<T> weak_ptr<T>::lock() const {
    // 伪代码示意原子过程：
    // 1. 原子地读取控制块中的引用计数
    // 2. 如果强引用计数 > 0，则原子地增加强引用计数
    // 3. 返回有效的 shared_ptr
    // 以上步骤在一个原子操作中完成
}
```

# shared_ptr

## 线程安全

**shared_ptr解决了引用计数的线程安全，但没有解决所有权的线程安全！！！**

shared_ptr的引用计数是原子操作，所以多个线程同时拷贝同一个shared_ptr对象是安全的，但是多个线程同时修改同一个shared_ptr对象（比如重置）则不是线程安全的。同时，访问shared_ptr指向的对象本身并不是线程安全的，除非对象内部有同步机制。

1. **引用计数是原子的**

```c++
// 多个线程同时复制同一个 shared_ptr 是线程安全的
std::shared_ptr<int> global_sp = std::make_shared<int>(42);

void thread_func() {
    auto local_sp = global_sp;  // ✓ 安全：引用计数原子增加
    // 每个线程有自己的 local_sp 副本
}
```

**安全**：`shared_ptr`的引用计数操作是原子的，多个线程同时复制/析构同一个 `shared_ptr`不会导致引用计数错误。

2. **多个线程修改同一个 shared_ptr 对象不是线程安全的**

```c++
std::shared_ptr<int> sp = std::make_shared<int>(100);

// 线程A
void thread_a() {
    sp = std::make_shared<int>(200);  // ✗ 不安全
}

// 线程B
void thread_b() {
    sp.reset(new int(300));  // ✗ 不安全
}
```

**不安全**：对同一个 `shared_ptr`对象进行写操作（赋值、reset 等）需要外部同步。

3. **多个线程读取同一个 shared_ptr 对象是安全的**

```c++
std::shared_ptr<int> sp = std::make_shared<int>(100);

// 线程A
void thread_a() {
    if (!sp) {  // ✓ 安全：只读
        // ...
    }
}

// 线程B
void thread_b() {
    auto copy = sp;  // ✓ 安全：读取并复制
}
```

**安全**：多个线程同时读取（包括复制构造）同一个 `shared_ptr`是线程安全的。

## 析构动作在创建时被捕获

在C++中，`shared_ptr`管理对象的生命周期是通过引用计数实现的。当创建一个`shared_ptr`时，我们不仅需要提供指向对象的指针，还可以选择性地提供一个删除器（deleter）。这个删除器是一个可调用对象，负责在引用计数归零时执行析构操作。**“析构动作在创建时被捕获”** 意味着删除器（即析构动作）是在创建`shared_ptr`时被绑定到该智能指针上的，而不是在对象类型中预先定义。

1. 虚析构不再是必需的

由于析构动作由`shared_ptr`的删除器负责，所以即使基类没有虚析构函数，当通过基类指针（如`shared_ptr<Base>`）管理派生类对象时，只要在创建时正确设置了删除器，就能正确调用派生类的析构函数。这是因为删除器在创建时已经知道如何正确释放对象。

```c++
struct Base {
    // 没有虚析构函数
    ~Base() { std::cout << "Base destructor\n"; }
};

struct Derived : Base {
    ~Derived() { std::cout << "Derived destructor\n"; }
};

int main() {
    // 使用shared_ptr管理Derived对象，即使Base没有虚析构，也能正确调用Derived的析构函数
    std::shared_ptr<Base> ptr(new Derived);
    // 当ptr离开作用域时，会正确调用Derived的析构函数，然后调用Base的析构函数
    // 这是因为在创建ptr时，shared_ptr内部捕获了正确的析构动作（即删除Derived对象）
    return 0;
}
```

2. `shared_ptr<void>`可以持有任何对象，而且能安全地释放

`void*`指针在C++中无法直接进行析构，因为`void`类型是不完整的。但是`shared_ptr<void>`可以持有任何类型的指针，因为它在创建时会捕获该类型的析构动作（删除器）。因此，即使静态类型是`void*`，在引用计数归零时，也会调用创建时捕获的析构动作来正确释放内存。

```c++
{
    // 创建一个int的shared_ptr，然后将其赋给shared_ptr<void>
    std::shared_ptr<int> i(new int(42));
    std::shared_ptr<void> v = i; // 正确，shared_ptr<void>可以持有任何类型

    // 也可以直接创建
    std::shared_ptr<void> p(new int(100));
    // 当p离开作用域时，int对象会被正确释放，因为创建p时捕获了int的删除器
}
```

3. `shared_ptr`对象可以安全地跨越模块边界

在Windows的DLL或Linux的共享库中，每个模块（动态库）有自己的堆内存分配器。如果在模块A中分配内存，在模块B中释放，可能导致未定义行为（因为分配和释放可能使用不同的堆管理器）。`shared_ptr`通过在创建时捕获删除器，可以确保使用正确的释放函数（即来自分配内存的那个模块的释放函数）。因此，从DLL返回一个`shared_ptr`到主程序，当引用计数归零时，会调用DLL中的释放函数，从而安全释放内存。

```c++
// 在DLL中
__declspec(dllexport) std::shared_ptr<int> create_shared_int() {
    return std::shared_ptr<int>(new int(42));
}

// 在主程序中
{
    auto ptr = create_shared_int(); // 从DLL获取shared_ptr
    // 当ptr离开作用域时，会调用DLL中的删除器来释放内存，避免跨模块释放问题
}
```

4. 二进制兼容性

如果动态库中某个类的实现改变了（例如对象大小发生变化），只要客户代码不直接访问该类的私有成员（即头文件中没有内联函数访问私有成员），并且客户代码通过工厂函数返回的`shared_ptr`来使用对象，那么旧的客户代码无需重新编译就能使用新版本的动态库。这是因为客户代码只通过指针和虚函数表与对象交互，而对象的构造和析构都由动态库中的工厂函数和`shared_ptr`的删除器负责，这些都在动态库内部实现。

```c++
// 动态库中的工厂函数
__declspec(dllexport) std::shared_ptr<Foo> createFoo();

// 客户代码
auto foo = createFoo(); // 返回shared_ptr<Foo>
// 即使Foo的内部实现改变，只要接口（虚函数）不变，客户代码无需重新编译
```

5. 析构动作可以定制

`shared_ptr`允许在创建时指定自定义的删除器，从而可以执行任意清理动作，而不仅仅是`delete`。这对于管理非`new`分配的资源（如文件指针、C风格数组、自定义内存池等）非常有用。

```c++
// 自定义删除器：使用free释放malloc分配的内存
struct FreeDeleter {
    void operator()(void* p) const { std::free(p); }
};

std::shared_ptr<int> intPtr(static_cast<int*>(std::malloc(sizeof(int))), FreeDeleter());

// 或者使用lambda表达式
std::shared_ptr<FILE> filePtr(std::fopen("test.txt", "r"), 
    [](FILE* fp) { if (fp) std::fclose(fp); });
```

## 



`*p2` 永远代表「a 对象本身」：

- 当你需要**值**时（如输出、赋值），它返回对象里的 10；
- 当你需要**地址**时（如 &(*p2)），它返回对象的内存地址 & a；
- 这是 C++ 表达式的 “上下文相关行为”，也是左值的核心特性。
