# 虚拟机字节码执行引擎

Java使用局部变量表来完成参数值到参数列表的传递过程，即实参到形参的传递。

如果被执行的是实力方法（没有被`static`修饰的方法），那局部变量中第0位索引的变量槽默认是用于传递方法所属对象实例的引用，在方法中可以通过`this`关键字来访问到这个隐藏参数。

为了节省栈帧空间，局部变量表中的变量槽是可以复用的，方法体中定义的变量，其作用于并不一定会覆盖整个方法体，如果当前字节码`PC`计数器的值已经超出了某个变量的作用域，那么这个变量对应的变量槽就可以交给其他变量来重用。

但是变量槽的重用会带来副作用：

```java
public static void main(String[] args) {
  byte[] placeholder = new byte[1024 * 1024];
  System.gc();
}
```

`GC`后并没有回收掉这64`MB`的内存。

这样还算符合逻辑，因为`placeholder`还在作用域之内，将代码修改：

```java
public static void main(String[] args) {
  {
    byte[] placeholder = new byte[1024 * 1024];
  }
  System.gc();
}
```

这样`placeholder`的作用域被限制在了花括号以内，从逻辑上来讲，在执行`GC`时，`placeholder`已经不可能再被访问了，但还是回收不掉。

再对代码进行修改：

```java
public static void main(String[] args) {
  {
    byte[] placeholder = new byte[1024 * 1024];
  }
  int a = 0;
  System.gc();
}
```

此时，`placeholder`可以被正确的回收。

原因如下：

这个问题的关键在于局部变量表中的变量槽是否还存有关于`placeholder`数组对象的引用。第一次修改中，代码虽然已经离开了`placeholder`的作用域，但是在此之后，在也没有发生过任何对局部变量表的读写操作，`placeholder`原本所占用的变量槽还没有被其他变量所复用，所以作为`GC Roots`一部分的局部变量表仍然保持着对它的关联。这种关联并没有被及时打断。


## 操作数栈

也常被称为操作栈，是一个`LIFO`栈，同局部变量表一样，操作数栈的最大深度也是在编译的时候被写到`Code`属性中。

操作栈中元素的数据类型必须与字节码的序列严格匹配，在编译程序代码的时候，编译器必须要严格保证这一点，这在类校验阶段的数据流分析中还要再次验证这一点。

> ​	`iadd`用于两个`int`类型数据的相加，这就要求靠近操作栈顶的两个元素必须为`int`类型，不能出现一个`long`和`float`使用`iadd`相加的情况。

java虚拟机被称为**“基于栈的执行引擎”**，里面的栈指的就是操作数栈。


## 动态连接

每个栈帧都包含一个指向运行是常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接。

`class`文件的常量池中存在大量的符号引用，字节码中的方法调用指令就以常量池里指向方法的符号引用作为参数，这些符号引用一部分会在类加载阶段或者第一次使用的时候就被转化为直接引用，这种转换称为**静态解析**。另一部分，将在每一次运行期间都转换为直接引用，这部分就称为**动态连接**。


## 方法返回地址

当一个方法执行后，只有两种方式退出这个方法：

* 执行引擎遇到任意一个方法返回的字节码指令，这个时候可能会有返回值传递给上层方法调用者，方法是否有返回值以及返回值的类型将根据遇到何种方法返回指令来决定，这种退出方法称为**正常调用完成**。
* 方法执行的过程中遇到了异常，并且这个异常没有在方法体内得到妥善处理。无论是java虚拟机内部产生的异常还是代码证使用`athrow`字节码指令产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种方式称为**异常调用完成**。一个方法使用异常完成出口的方式退出，不会给它的上层调用者提供返回值。

无论采用哪种方式退出，在方法退出后，都必须返回到最初方法被调用时的位置，程序才能继续执行，方法返回时可能需要在栈帧中保存一些信息，用来帮助恢复它的上层主调用方法的执行状态。

* 正常退出：主调方法的`PC`计数值就可以作为返回地址，栈帧中很可能保存这个计数值。
* 异常退出：返回地址要根据异常处理器表来确定，栈帧中就一般不会保存这部分信息。



## 方法调用

方法调用并不等于方法中的代码被执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法），暂时还未涉及到方法内部呢的具体运行过程。

​	`Class`文件的编译过程中不包含传统程序语言编译的连接阶段，**一切方法调用在`class`文件中不过是符号引用，而不是方法在实际运行时的内存布局中的入口地址。**

### 解析

在类加载的解析阶段，会将其中一部分符号引用转换为直接引用，这种解析能够成立的前提是：**方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期间是不可改变的。**

> 调用目标在程序代码写好、编译器进行编译的那一刻就已经确定下来。这类方法的调用被称为解析。

java中符合**编译器可知，运行期不变**这个要求的方法，主要有**静态方法**和**私有方法**两类，前者与类型直接关联，后者在外部不可被访问，这两种方法各自的特点决定**它们不可能通过继承或者别的方法重写出其他版本，因此它们都适合在类加载阶段进行解析。**

调用不同类型的方法，Java虚拟机支持五种方法调用字节码：

* `invokestatic`：调用静态方法。
* `invokespecial`：调用实例构造器`<init>`方法，私有方法和父类中的方法。
* `invokevirtual`：调用所有的虚方法。
* `invokeinterface`：调用接口方法，会在运行时再确定一个是该接口的对象。
* `invokedynamic`：现在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

> 前四条的分派逻辑固化在java虚拟机内部，而第五条的分派逻辑则由用户设定的引导方法来决定。

只要能被`invokestatic`和`invokespecial`指令调用的方法，都可以在解析阶段中确定唯一的调用版本，java语言里符合这个条件的方法有**静态方法**、**私有方法**、**实例构造器**、**父类方法**四种，再加上被`final`修饰的方法（被`invokevirtual`修饰），这些方法被称为**非虚方法**，一直相反，则被称为**虚方法**。

解析调用一定是个静态的过程，在编译期间就完全确定，在**类加载的解析阶段就会把涉及的符号引用全部转变为明确的直接引用，不必延迟到运行期间再去完成。**


### 分派

分派调用揭示了多态性特征的一些最基本的体现，如“重载“和”重写“在JVM中的实现。

#### 静态分派

静态分派应该归于类加载中的解析过程。

```java
public class StaticDispatch {
    static abstract class Human {
    }

    static class Man extends Human {
    }

    static class Woman extends Human {
    }

    public void sayHello(Human guy) {
        System.out.println("hello guy!");
    }
    public void sayHello(Man guy) {
        System.out.println("hello man");
    }
    public void sayHello(Woman guy) {
        System.out.println("hello woman");
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}
```

结果都是输出`hello guy!`。

上面代码中`Human`称为变量的**`静态类型(static type)`**，或者叫**`外观类型(Apparent Type)`**，而`Man`则称之为**`运行时类型(Runtime Type)`**。

`静态类型`与`实际类型`在程序中都可能发生变化，区别是`静态类型`的变化只在使用时发生变化，变量本身的`静态类型`是不会发生改变的，并且最终的`静态类型`在编译期间是可知的。而`实际类型`变化的结果只能在运行期间才能确定，编译器在编译程序时，并不知道一个对象的实际类型。

example：

```java
// 实际类型变化，只能等到运行期才能确定
Human human = (new Random()).nextBoolean() ? new Man() : new Woman();

/**
	静态类型变化
	编译期间就可以知道它的静态类型为Human，此处即使发生了改变，也知道它是Man或者Woman，可以确定
**/
sr.sayHello((Man) human);
sr.sayHello((Woman) human);
```



在`StaticDispatch`中，定义了两个`静态类型`相同，`实际类型`不同的变量，但是JVM（准确的说是编译器）在重载时是通过参数的`静态类型`而不是`实际类型`作为判定依据的。由于`静态类型`在编译期间可知，所以在编译阶段`javac`编译器就根据参数的`静态类型`决定了会使用那个重载版本。*所以选择了`sayHello(Human)`这个方法*。

**所有依赖静态类型来决定方法执行版本的分派动作，都被称为`静态分派`**，`静态分派`最典型应用表现就是**方法重载**。

`静态分派`发生在编译阶段，因此确定`静态分派`的动作不是由虚拟机来执行的，所以它可以被归为**解析**，而不是**分派**。

`Javac`编译器虽然能确定出方法的重载版本，但是在许多情况下，这个重载版本并不是“唯一”的，所以只能确定一个“相对更合适”的版本。因为字面量天生就是模糊的。

example：

```java
public class Overload {
    public static void sayHello(Object arg) {
        System.out.println("hello obj");
    }

    public static void sayHello(int arg) {
        System.out.println("hello int");
    }

    public static void sayHello(long arg) {
        System.out.println("hello long");
    }

    public static void sayHello(Character arg) {
        System.out.println("hello Character");
    }

    public static void sayHello(char arg) {
        System.out.println("hello char");
    }

    public static void sayHello(char... arg) {
        System.out.println("hello char ...");
    }

    public static void sayHello(Serializable arg) {
        System.out.println("hello Serializable");
    }

    public static void main(String[] args) {
        sayHello('a');
    }
}
```

这个例子中，会发生自动转型，顺序为：`char > int > long > float > double`。但是不会匹配到`byte`和`short`，因为向下转型是不安全的。

将这些都注释掉之后，就会去调用`sayHello(Character)`，再注释后就会调用`sayHello(Serializable)`，因为它转型会发现都不存在，只有装箱类的父接口，那么转型为`Character`实现的接口或者父类，但是`Character`也实现了`Comparabele`，它们是平级的，如果`main()`，出现了`sayHello(Comparable)`，那么编译器就无法确定要自动转型为哪种类型，就会拒绝编译。

然后，继续注释，可以知道调用顺序依次为`sayHello(Object)`与`sayHello(char...)`，可见变长参数重载优先级是最低的。

> ​	但是变长参数不会向其他类型转换，此处就不会转向`int...`。


#### 动态分派

它与多态性的另一个重要体现——重写有着密切的关联。

exmple：

```java
public class DynamicDispatch {
    static abstract class Human {
        protected abstract void sayHello();
    }

    static class Man extends Human {
        @Override
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }

    static class Woman extends Human {
        @Override
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}
```

其字节码文件：

```java
Compiled from "DynamicDispatch.java"
public class com.wx.jvm.DynamicDispatch {
  public com.wx.jvm.DynamicDispatch();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class com/wx/jvm/DynamicDispatch$Man
       3: dup
       4: invokespecial #3                  // Method com/wx/jvm/DynamicDispatch$Man."<init>":()V
       7: astore_1
       8: new           #4                  // class com/wx/jvm/DynamicDispatch$Woman
      11: dup
      12: invokespecial #5                  // Method com/wx/jvm/DynamicDispatch$Woman."<init>":()V
      15: astore_2
      16: aload_1
      17: invokevirtual #6                  // Method com/wx/jvm/DynamicDispatch$Human.sayHello:()V
      20: aload_2
      21: invokevirtual #6                  // Method com/wx/jvm/DynamicDispatch$Human.sayHello:()V
      24: new           #4                  // class com/wx/jvm/DynamicDispatch$Woman
      27: dup
      28: invokespecial #5                  // Method com/wx/jvm/DynamicDispatch$Woman."<init>":()V
      31: astore_1
      32: aload_1
      33: invokevirtual #6                  // Method com/wx/jvm/DynamicDispatch$Human.sayHello:()V
      36: return
}
```

关注`main`方法：

其中0~15行为字节码的准备动作，16行与20行是`aload_1`将创建的两个对象压入栈顶，它们都是将要执行`sayHello`方法的所有者，称为接收者`Receiver`；17行与21行是`invokevirtual`，是真正执行目标方法的指令。

`invokevirtual`指令的大概步骤：

1. 找到操作数栈顶的第一个元素所指向的对象的`实际类型`，记为C。
2. 如果在类型C中找到了常量中的描述符和简单名称都符合的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找过程结束；不通过，则抛出`java.lang.IllegalAccessError`异常。
3. 否则，按照继承关系自底向上依次对C的父类重复步骤2。
4. 如果始终没找到合适的方法，则抛出`java.lang.AbstractMethodError`。

正是因为`invokevirtual`指令执行的第一步就是在运行期确定接收者的`实际类型`，所以两次调用中的`invokevirtual`指令并不是把常量池中的方法的符号引用解析道直接引用上就结束了，还会根据方法接收者的`实际类型`来选择方法版本，这就是**java中重写的本质**。

这种多态性的根源在于虚方法调用指令`invokevirtual`的执行逻辑，所以这只对方法有效，对字段无效，也就是说**java中只有虚方法存在，字段永远不可能为虚，字段永远不可能参与躲在，哪个类的方法访问某个名字的字段时，该名字指的就是这个类能看到的那个字段。**

当子类声明了与父类同名的字段时，虽然在子类的内存中两个字段都会存在，但是子类的字段会遮蔽父类的同名字段。

example：

```java
public class FieldHasNoPolymorphic {
    static class Father {
        public int money = 1;
        public void showMeTheMoney() {
            System.out.println("I am Father, i have " + money);
        }
        public Father() {
            money = 2;
            showMeTheMoney();
        }
    }
    static class Son extends Father {
        public int money = 3;
        public Son() {
            money = 4;
            showMeTheMoney();
        }
        public void showMeTheMoney() {
            System.out.println("I am Son, i have " + money);
        }
    }

    public static void main(String[] args) {
        Father f = new Son();
        System.out.println("This gay has " + f.money);
    }
}

// 输出结果
I am Son, i have 0
I am Son, i have 4
This gay has 2
```



1. `Son`在创建时，首先隐式调用了父类的构造函数，而`Father`中对`showMeTheMoney()`的调用，是一次虚方法调用，实际执行的版本是`Son::showMeTheMoney()`。而此时`Son`中的`money`字段只分配了内存，还没有初始化所以输出0。
2. 之后调用`Son`自己的构造函数，调用`showMeTheMoney()`时，输出4。
3. 由于字段不存在动态分派，所以`f.money`是通过`静态类型`访问到父类中的`money`。


#### 单分派与多分派

方法的接收者与方法参数的参数统称为方法的宗量，根据分派基于多少种宗量，可以讲分派划分为单分派和多分派。

```java
public class Dispatch {
    static class QQ {}
    static class _360 {}
    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }
        public void hardChoice(_360 arg) {
            System.out.println("father choose 360");
        }
    }

    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }
        public void hardChoice(_360 arg) {
            System.out.println("son choose 360");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }
}

// 输出结果
father choose 360
son choose qq
```

**编译阶段的选择过程即`静态分派`方法选择依据：**

1. 静态类型是`Father`还是`Son`。
2. 方法参数是`QQ`还是`_360`。

> 这次选择结果的最终产物是产生了两条`invokevirtual`指令，分别为`Father::hardChoice(QQ)`与`Father::hardChoice(_360)`。

因为是根据两个宗量（方法接收者与方法参数）进行选择，所以**java属于`静态多分派`类型**。

**运行阶段的选择过程即`动态分派`方法选择依据：**

在执行`son.hardChoice(new QQ())`时（准确来说是执行这行代码对应的`invokevirtual`），由于编译器已经决定目标方法的前面是`hardChoice(QQ)`，此时虚拟机并不关心这个`QQ`是它的那个子类，**因为此时参数的静态类型、实际类型对方法的选择不会构成任何影响（它只看到了`QQ`），唯一可以影响的是该方法`接收者`的实际类型是`Father`还是`Son`。**

**因为只有一个宗量作为选择依据，所以java是动态单分派类型。**


所以**java是一门静态多分派、动态单分派的语言。**

