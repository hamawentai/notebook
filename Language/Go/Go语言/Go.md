# 基本结构和基本数据类型

## 包

每个Go文件都属于且仅属于一个包。一个包可以由许多.go为扩展名的源文件组成，所以**文件名和包名一般来说都是不相同的。**

* 当标识符（变量、常量、类型、函数名、结构字段等等）以一个大写字母开头，则可以被外部包代码使用（public）
* 当标识符（变量、常量、类型、函数名、结构字段等等）以一个小写字母开头，则不可以被外部包代码使用（private）

如果导入了一个包却没有使用它，则会报错，Go的格言：“没有不必要的代码”。？？？

`{`必须与声明放在同一行。

```go
func f1(a int) {
  /** do something **/
} // 正确

func f1(a int) {} // 正确
  
func f1(a int) 
{
  
} // 错误
```

## 类型

Go中的空值为Nil，等同于java中的null。

类型转换：

类型B的值 = 类型B(类型A的值)

```go
a = typeB(valA);
```

Go中不允许不同类型之间的混用，但是对于常量的类型限制非常少，所以常量之间是可以混用的

```go
var a int
var b int32
a = 15
b = a + a // 错误：不允许混用
b = b + 5 // 5
```

### 函数定义

```go
func FunctionName (a typeA, b,c typeB) (t1 typeT1, t2, t3 typeT3) {
  /** do something **/
}
```

一个函数可以由多个返回值，返回类型之间需要逗号分隔，当参数列表长度大于1时，需要使用`()`来将它们扩起来。

### 别名

```go
type INT int
var a INT = 5
```

*类似c++中的typedef。*

也可以使用**因式分解**的定义方式：

```go
type (
	IZ int
  FZ float64
  STR string
)
```

每个值必须在经过编译后属于某一个类型（编译器必须能够推断出所有值的类型），因为Go是一种静态类型的语言。

## `Go`程序的一般结构

* 完成对包的import之后，开始对常量、变量和类型的定义或声明
* 如果存在**init**函数的话，则对该函数进行定义。（每个函数该函数的包都会首先执行这个函数）
* 如果当前包时main包，则定义main函数
* 然后定义其余的函数
  * 首先是类型方法
  * 按照main函数中先后调用顺序来定义相关函数，如果有很多函数，则可以按照字母顺序来今昔排序

执行顺序：

* 导入被main包包含的所有包
  * 如果被包含的包又包含了其他包，那么DFS的导入包（每种包只会被导入一次）
* 然后执行init函数如果有的话，再是main函数

## 常量

```go
const identifier [type] = value
const Pi = 3.1415926
```

可以省略type，编译器可以根据变量的值来推断其类型。

* 显示类型定义

  ```go
  const b string = "abc"
  ```

* 隐式类型定义

  ```go
  const b = "abc"
  ```

未定义类型的常量会在必要时刻根据上下文来获取相关类型。

```go
var n int
f(n + 5)// 无类型的数字常量“5”，它的类型在此处由n的推导变为了int
```

所有用于计算的值必须在编译器见就能被获取到。

```go
const c1 = geNum() // 错误
```

在编译期间自定义的函数均属于未知，所以无法用于常量的复制，但是内置函数可以使用，如：len()。

### 枚举类型

常量也可以用作枚举。

```go
const (
	A = 0
  B = 1
  C = 2
)
// 或者使用iota
const (
	A = iota
  B
  C
)
// 此时A、B、C分别是0、1、2
```

iota也可以用在表达式中如：`ioat + 10`，每遇到一次const关键字，ioat就重置为0。

## 变量

声明变量：

```go
var identifier type
```

也可以使用因式分解的形式：

```go
var (
	a int
  b bool
  str string
)
```

这种方式一般用于声明全局变量。

且在Go也存在作用域覆盖的情况，即局部变量会覆盖全局变量的值。

若是直接给变量赋值，那么可以省略掉类型，Go编译器可以在编译时期完成推断过程。

```go
var a = 15 // 此时a被推断为int
// 若是需要特定的int，那么还是需要显式的指出类型
var b int64 = 10
```

上面这种写法主要用于声明全局变量，若是函数体内的局部变量，则可以使用`:=`：

```go
a := 16
b := "Hello World"
```

## `init`函数

它不能够被人为调用，**而是在每个包完成初始化后自动执行，并且执行优先级比main函数要高。**

每个源文件都可以包含一个或多个init函数。初始化总是以单线程执行，并且按照包的依赖关系顺序执行。

`init`函数也经常被用在当一个程序开始之前调用后台执行的`goroutine`。

## 字符串

* 解释字符串：转义字符能在其中生效，用双引号括起来。

* 非解释字符串：转义字符能在其中会失效，用单引号括起来。

  ```go
  str := `Hello World\n` // \n会被输出出来
  ```

和c/c++不同，Go中的字符串是根据长度限定，而非特殊字符`\0`。

Go的字符串也是可以通过`[]`来获取其中的字符，但是不能对其取地址`&str[i]`。

## 指针

Go提供了控制数据结构的**指针**，**但是不能进行指针运算**，即对指针加减。

```go
var a *int
b := 10
a = &b
a++ // 错误：不能对指针进行运算
```



# 控制结构

* if-else

  ```go
  if condition {
    
  } // 不需要()
  ```

* switch

  每个case后面不需要break，如果需要这种效果的话，可以使用`fallthrough`

  ```go
  switch var1 {
    case val1:
     	// ...
    case val2:
    	// ...
    case val3, val4, val5:
    	// ...
    	fallthrough
    default:
    	// ...
  }
  ```

* for

  * for 初始化语句; 条件语句 ; 修饰语句 {}

    ```go
    for i:=0; i<5; ++i {
      // ...
    }
    ```

  * 无头部的条件判断语句。 for 条件语句 {}

    ```go
    for i>=0 {
      // ...
    }
    ```

  * for-range

    for index, val := range list {}

    ```go
    str := "Hello World"
    for i, ch := str {
      // ...
    }
    ```

* break、continue：没啥特殊的

* goto？go8！

# 函数

Go里面有三种类型的函数：

* 普通带名字的函数
* 匿名函数或者lambda函数
* 方法

函数定义时，形参一般都是有名字，不过也可以定义没有形参名字的函数，只有相应的形参类型。

```go
func f(int, int)
```

这些函数被称为`niladic`函数？

## 值传递与引用传递

Go默认采用**按值传递**参数。但是像切片`slice`、字典`map`、接口`interface`、通道`channel`这样的引用类型都是默认引用传递的。

### 命名的返回值

返回值分为：

* 非命名返回值
* 命名返回值

```go
func f1(i int)(int int) { // 非命名返回值
  return i, i+2;
}

func f2(i int)(x1 x2 int) { // 命名返回值
  x1 = i;
  x2 = i + 2;
  return
}
// 对于不需要使用的返回值可以使用空白符替代
_, b = f1(10);
```

## 传递参数

### 变长参数

最后一个参数是`...type`时就可以处理一个变长的参数。

```go
func f1(a,b, arg...int) {
  for i,x := arg {
    // ...
  }
}
```

其他解决方法：

* 使用结构体

  ```go
  type MyStruct struct {
    par1 int,
    par2 string,
    par3 float32
  }
  
  F1(MyStruct {})
  F1(MyStruct {1, "hello", 3.14})
  ```

* 使用空接口

  如果一个变长参数的类型没有被指定，则可以使用默认的空接口`interface{}`。

  ```go
  func f1(vals ...interface{}) {
    for _ ,val := range vals {
      switch v := value.(type) {
        case int: // ...
        case string: // ...
        case float32:	// ...
        // ...
      }
    }
  }
  ```



## `defer`和追踪

`defer`允许推迟到函数返回之前（或任意位置执行`return`语句之后，因为`return`语句同样可以包含一些操作，而不是单纯地返回某个值）一刻才执行某个语句或函数。

**类似于Java的`finally`。**

```go
func f1() {
  fmt.Println("T2 before")
	defer fmt.Println("defer")
	fmt.Println("T2 after")
}
// 结果：
/**
T2 before
T2 after
defer
**/
```

当有多个defer行为被注册时，它们会以逆序执行（类似栈）。

## 函数作为参数

```go
func add(a, b int) int {
	return a + b
}

func useFunc(a, b int, f func(int, int) int) {
	fmt.Println(f(a, b+10))
}

func T3(a, b int) {
	useFunc(a, b, add);
}
```

## 闭包

匿名函数：`func(x, y int) int {return x+y}`

```go
fp := func(x, y int) int {
  return x+y
}
fp(1, 2);
```

可以将函数作为返回值

```go
func f1(a int) (func(b int) int)
func f1(a int) (func(b int) int) {
  return func(b int) int {
    a + b
  }
}
```



# 数组

类似于python的切片

数据的长度属于它的类型。如`[10]int`与`[5]int`并不是一种类型。

声明格式为：

```go
var arr1 [10]int
```

Go的数组是一种**值类型**（并不是c++中的指针），所以可以通过`new()`来创建

```go
var arr2 = new [5]int
```

其中arr1的类型是`[10]int`，而arr2的类型是`*[10]int`。

## 多维数组

Go中的多维数组是一个矩阵（唯一例外的是切片的数组）。

**将数组传递给函数**

类似C++，为避免值传递的拷贝：

* 传递数组的指针

  ```go
  func f1(a *[3]int) int {
    sum := 0
    for _,v := range a {
      sum += v
    }
    return sum
  }
  var arr = [3]int{1,2,3}
  var arr = [...]int{1,2,3}
  var arr = [3]int{1:1, 2:2} // index:value
  f1(&arr)
  ```

* 传递数组的切片

## 切片

切片是对数组一个连续片段的引用，所以切片是一个**引用类型**。

几个属性：

* `len()`：切片元素实际占用的元素长度
* `cap()`：切片的全部长度=`len()`+剩余部分长度

```go
a[start:end] // 左闭右开的区间
```

<img src="pic/1.png" style="zoom:50%;" />

