# 第二章 Java内存区域与内存溢出异常 

## 运行时数据区域

![](https://raw.githubusercontent.com/yxcoder1997/PictureBed/master/img/运行时数据区.png)

Java虚拟机在执行Java程序时，会把它所管理的内存划分成不同的数据区，如下图所示

![1594303092760.png](https://github.com/yxcoder1997/PictureBed/blob/master/img/1594303092760.png?raw=true)

### 程序计数器

程序计数器（Program Counter Register）只占用较小的内存空间， 它可以看作是**当前线程所执行的字节码的行号指示器**。 字节码解释器工作时就是**通过改变这个计数器的值来选取下一条需要执行的字节码指令**， 它是程序控制流的指示器， 分支、 循环、 跳转、 异常处理、 线程恢复等**基础功能都需要依赖这个计数器来完成** 。

由于Java的多线程是每个线程轮流分配处理时间来完成的，所以在每一时刻处理器只会执行一条线程中的指令 。为了让在切换线程后可以恢复到正确的执行位置，**每一个线程都一个独立的程序计数器**，各条线程之间计数器互不影响， 独立存储， 我们称这类内存区域为**“线程私有”的内存**。 

### Java虚拟机栈

虚拟机栈描述的是**Java方法执行的线程内存模型**： 每个方法被执行的时候， Java虚拟机都会同步创建一个栈帧（Stack Frame） 用于存储局部变量表、 操作数栈、 动态连接、 方法出口等信息。 每一个方法被调用直至执行完毕的过程， 就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。 

#### 局部变量表

存在着以下3种类型变量

- 基本数据类型：8个基本数据
- 对象引用：引用类型
- returnAddress类型：指向了一个字节码指令地址

这些数据类型在局部变量表中的存储空间以**局部变量槽**（Slot） 来表示， 其中64位长度的**long和double**类型的数据会**占用两个变量槽**， 其余的数据类型只占用一个。 

局部变量表所需要的内存空间在编译期间完成，当进入一个方法时， 这个方法需要在栈帧中分配多大的局部变量空间是完全确定的， 在方法运行期间不会改变局部变量表的大小。 

### 本地方法栈

本地方法栈是为虚拟机使用到的**本地方法服务**。

**区别**：虚拟机栈为虚拟机执行**Java方法**（也就是**字节码**） 服务 

### Java堆

Java堆是被所有线程共享的一块内存区域， **在虚拟机启动时创建**。 此内存区域的唯一目的就是存放对象实例 ，J**ava对象实例都分配在堆上** ，但由于即时编译技术的进步， 尤其是逃逸分析技术的日渐强大 ，变得不那么绝对。

### 方法区

方法区与Java堆一样， 是各个线程共享的内存区域， 它用于存储已被虚拟机加载的类型信息、 常量、 静态变量、 即时编译器编译后的代码缓存等数据 。

JDK8以前，很多人更愿意把方法区称为永久代，或者把两个混为一谈。本质上两者并不是等价的。

### 运行时常量池

运行时常量池是方法区的一部分。 Class文件中除了有类的版本、 字段、 方法、 接口等描述信息外， 还有一项信息是常量池表，**用于存放编译期生成的各种字面量与符号引用**， 这部分内容将在类加载后存放到方法区的运行时常量池中 。

运行时常量池相对于Class文件常量池的另外一个重要特征是**具备动态性**， Java语言并不要求常量一定只有编译期才能产生， 也就是说， 并非预置入Class文件中常量池的内容才能进入方法区运行时常量池， 运行期间也可以将新的常量放入池中， 这种特性被开发人员利用得比较多的便是String类的intern()方法。 

### 直接内存

直接内存不是虚拟机运行时数据区的一部分。

但这部分内存也会频繁使用，并出现OutofMemoryError异常。

JDK1.4新加入了NIO（New Input/Output）类，引入了一种基于通道与缓冲区的I/O方式，它使用Native函数库直接分配堆外内存。

显然， 本机直接内存的分配不会受到Java堆大小的限制， 但是， 既然是内存， 则肯定还是会受到本机总内存（包括物理内存、 SWAP分区或者分页文件） 大小以及处理器寻址空间的限制。

## HotSpot虚拟机对象探秘 

![](https://raw.githubusercontent.com/yxcoder1997/PictureBed/master/img/类与实例.png)

### 对象的创建

在Java语言中，创建对象通常利用new关键字，然而在虚拟机中，对象的创建是个什么过程呢？

#### 类加载检查

当虚拟机解析.class文件，遇到new指令时，首先去检查常量池中是否有这个类的符号引用，并且检查这个符号引用代表的类是否已被加载、 解析和初始化过。 如果没有， 那必须先执行相应的类加载过程。

#### 为新生对象分配内存

对象所需内存的大小在类加载完成后便可完全确定，接下来从堆中划分一块对应大小的内存空间给新的对象。分配堆中内存有两种方式：

- **指针碰撞**

  假设Java堆中内存是**绝对规整**的， 所有被使用过的内存都被放在一边， 空闲的内存被放在另一边， 中间放着一个指针作为分界点的指示器， 那所分配内存就仅仅是把那个指针向空闲空间方向挪动一段与对象大小相等的距离，这种分配方式称为“**指针碰撞**”。

- **空闲列表**

  如果Java堆中的内存并**不是规整**的， 已被使用的内存和空闲的内存相互交错在一起， 那就没有办法简单地进行指针碰撞了， 虚拟机就必须维护一个列表， 记录上哪些内存块是可用的， 在分配的时候从列表中找到一块足够大的空间划分给对象实例， 并更新列表上的记录， 这种分配方式称为**“空闲列表”** 。 

选择哪种分配方式由Java堆是否规整决定， 而Java堆是否规整又由所采用的垃圾收集器是否带有**空间压缩整理（Compact） 的能力决定**。 

#### 初始化

虚拟机必须将分配到的内存空间（但不包括对象头） 都初始化为零值 ，为对象头设置信息，虚拟机状态不同会有不同的设置。

以上所有工作都完成之后 ：

- 虚拟机视角：一个新的对象产生
- Java程序视角：对象创建刚刚开始，构造函数没有执行。

### 对象的内存布局

在 HotSpot 虚拟机中，对象的内存布局分为以下 3 块区域：

- 对象头（Header）
- 实例数据（Instance Data）
- 对齐填充（Padding）

![20200716125958.png](https://github.com/yxcoder1997/PictureBed/blob/master/img/20200716125958.png?raw=true)

#### 对象头

包含两类信息

- **用于存储对象自身的运行时数据**

  - 哈希码

  - GC分代年龄

  - 锁状态标志

  - 线程持有的锁

  - 偏向线程ID 

  - 偏向时间戳

    在32位和64位虚拟机中分别位32bit和64bit，官方称为"Mark Word",对象头里的信息是与对象自身定义的数据无关的额外存储成本， 考虑到虚拟机的空间效率， Mark Word被设计成一个有着动态定义的数据结构， 以便在极小的空间内存储尽量多的数据， 根据对象的状态复用自己的存储空间。 

    ![20200716140551.png](https://github.com/yxcoder1997/PictureBed/blob/master/img/20200716140551.png?raw=true)

- **类型指针**

  即对象指向它的类型元数据的指针 ，通过指针能确定对象属于哪个类

#### 实例数据

它是对象真正存储的有效信息，即成员变量的值，无论是从父类继承下来的， 还是在子类中定义的字段都必须记录起来。  

#### 对齐填充

于HotSpot虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍， 换句话说就是任何对象的大小都必须是8字节的整数倍 。如果对象实例数据部分没有对齐的话， 就需要通过对齐填充来补全。 

### 对象的访问定位

由于reference类型在《Java虚拟机规范》 里面只规定了它是一个指向对象的引用， 并没有定义这个引用应该通过什么方式去定位、 访问到堆中对象的具体位置， 所以对象访问方式也是由虚拟机实现而定的， 主流的访问方式主要有使用句柄和直接指针两种：

#### 通过句柄访问对象 

在Java堆中，专门开辟出一块内存，称为句柄池，reference中存储的就是对象的句柄地址，句柄池存放着对象实例数据与类型数据各自具体的地址信息 。

![20200716141714.png](https://github.com/yxcoder1997/PictureBed/blob/master/img/20200716141714.png?raw=true)

**优点**：reference中存储的是稳定句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为） 时只会改变句柄中的实例数据指针， 而reference本身不需要被修改。 

#### 通过直接指针访问对象 

reference中存储的直接就是对象地址， 如果只是访问对象本身的话， 就不需要多一次间接访问的开销。

![20200716141942.png](https://github.com/yxcoder1997/PictureBed/blob/master/img/20200716141942.png?raw=true)

**优点**：使用直接指针来访问最大的好处就是速度更快， 它节省了一次指针定位的时间开销， 由于对象访问在Java中非常频繁， 因此这类开销积少成多也是一项极为可观的执行成本 。