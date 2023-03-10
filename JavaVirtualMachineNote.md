# 自动内存管理

[![JVM-data-area.png](https://i.postimg.cc/Y9PKwLZD/JVM-data-area.png)](https://postimg.cc/phK1fr2K)

## 程序计数器(Program Counter Register)

由于JVM的多线程是通过线程轮流切换、分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，**每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储**。

如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是本地(Native)方法，这个计数器值则应为空(Undefined)。此内存区域是**唯一一个在《Java虚拟机规范》中没有规定任何`OutOfMemoryError`情况的区域**。

## Java虚拟机栈(Java Virtual Machine Stack)

Java虚拟机栈也是线程私有的，它的生命周期与线程相同。每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧(Stack Frame)用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

[![stack-frame.png](https://i.postimg.cc/bvTSLpWw/stack-frame.png)](https://postimg.cc/cKvJLydp)

这里只讨论虚拟机栈中的局部变量表部分。局部变量表存放了编译期可知的各种基本数据类型(Primitive types)、对象引用（Reference类型，它并不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。

这些数据类型在局部变量表中的存储空间以局部变量槽(Slot)来表示，其中64位长度的`long`和`double`类型会占用两个变量槽，其余的数据类型只占用一个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间（指slot的个数）是完全确定的，在方法运行期间不会改变局部变量表的大小（指slot的个数）。

如果线程请求的栈深度大于虚拟机所允许的深度，将抛出`StackOverflowError`异常；如果Java虚拟机栈容量可以动态扩展，当栈扩展时无法申请到足够的内存会抛出`OutOfMemoryError`异常。

## 本地方法栈

虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的本地方法服务。HotSpot虚拟机等直接将本地方法栈和虚拟机栈合二为一。其抛出异常也与虚拟机栈抛出的一致。

## Java堆(Java Heap)

虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例和数组。Java世界里“几乎”所有的对象实例和数组都在这里分配内存。（因此，基本数据类型不一定保存在栈中，作为数组元素或者类的成员变量时，在不开启逃逸分析进行优化的情况下，都被保存在堆中。）

根据《Java虚拟机规范》的归档，Java堆可以处于物理上不连续的内存空间中，但在逻辑上它应该被视为连续的。但对于大对象（典型的如数组对象），多数虚拟机实现出于实现简单、存储高效的考虑，很可能会要求连续的内存空间。

当前主流虚拟机的堆的大小都是可以扩展的。如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展时，虚拟机将会抛出`OutOfMemoryError`异常。

## 方法区(Method Area)

方法区也是各个线程共享的内存区域，用于存储已被虚拟机加载的类元信息（类型元数据）、运行时常量池、即时编译器编译后的代码缓存等数据。除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，甚至还可以选择不实现垃圾收集。下图是JDK8 HotSpot虚拟机基于元空间-本地内存的方法区示意图：

[![method-area.png](https://i.postimg.cc/YCPs0Bbj/method-area.png)](https://postimg.cc/pmDZc4Nt)

### 类元信息（类型元数据）

+ 类型信息

  对每个加载的类型(class, interface, enum, annotation)，JVM方法区中存储以下类型信息：
1. 这个类型的完整有效名称（包名.类名）
2. 这个类型直接父类的完整有效名（对于interface或者是java.lang.Object，都没有父类）
3. 这个类型的修饰符（public, abstract, final的某个子集）
4. 这个类型直接实现的接口的一个有序列表

+ 域(Field)信息

1. JVM必须在方法区中保存类型的所有域的相关信息以及域的声明顺序
2. 域的相关信息，包括：域名称、域类型、域修饰符(public, private, protected, final, volatile, transient的某个子集)

+ 方法(Method)信息

  JVM必须保存所有方法的以下信息，和域信息一样，声明顺序也包括在内：
1. 方法名称
2. 方法的返回类型
3. 方法参数的数量和类型（按顺序）
4. 方法的修饰符
5. 方法的字节码、操作数栈、局部变量表及大小（abstract和native方法除外）
6. 异常表（abstract和native方法除外)：每个异常处理的开始位置、结束位置、代码处理器在程序计数器中的偏移地址、被捕获的异常类的常量池索引

+ 方法表

  方法表是一组对类实例方法的直接引用（包括从父类继承的方法）。JVM可以通过方法表快速激活实例方法，从而提高访问效率。

+ 类加载器的引用

  JVM必须知道一个类型是由启动加载器加载的还是由用户类加载器加载的。如果一个类型是由用户类加载器加载的，那么JVM会将这个类加载器的一个引用作为类型信息的一部分保存在方法区中。JVM在动态连接的时候需要这个信息。当解析一个类型到另一个类型的引用的时候，JVM需要保证这两个类型的类加载器是相同的。这对JVM区分名字空间的方式是至关重要的。

+ Class类实例的引用

  JVM为每个加载的类都创建一个java.lang.Class的实例（存储在堆上）。而JVM必须以某种方式把Class的这个实例和存储在方法区中的类型数据（类的元数据）联系起来，因此，类的元数据里面保存了一个Class对象的引用。

在JDK7及以前，习惯上把方法区称为永久代，JDK8以后，使用元空间取代了永久代。元空间与永久代最大的区别在于，元空间不在虚拟机设置的内存当中，而是使用本地内存。

如果方法区无法满足新的内存分配需求时，将抛出`OutOfMemoryError`异常。

### 运行时常量池(Runtime Constant Pool)

**运行时常量池是方法区的一部分**。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表(Constant Pool Table)，用于存放编译器生成的各种**字面量**与**符号引用**，这部分内容将在类加载后放到方法区的运行时常量池中。注意，Java并不要求常量一定只有编译期才能产生，并非预置入Class文件中常量池表的内容才能进入运行时常量池，运行期间也可以将新的常量放入池中，例如String类的intern()方法。

+ 字面量

字面量是用于表达源代码中一个固定值的表示法，包括文本字符串、被声明为final的常量值、基本数据类型值等
```Java
int a = 10; // 10为字面量
String b = "Hello World!"; // "Hello World!"是字面量
final int b = 20; // b为常量，20为常量值
```
+ 符号引用

包括类和接口的全限定名、字段的名称和描述符、方法的名称和描述符。

显然运行时常量池受到方法区内存的限制，当常量池无法再申请到内存时会抛出`OutOfMemoryError`异常。

## 直接内存(Direct Memory)

直接内存并不是虚拟机运行时数据区的一部分，但是这部分内存也被频繁使用，而且也可能导致`OutOfMemoryError`异常出现。

NIO类引入了一种基于通道(Channel)和缓冲区(Buffer)的IO方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样避免了在Java堆和Native堆中来回复制数据而对性能的影响。

直接内存的分配不会受到Java堆大小的限制，但会受到本机总内存大小以及处理器寻址空间的限制。如果在配置虚拟机参数时忽略掉直接内存，使得各个区域内存综合大于物理内存限制，从而导致动态扩展时出现`OutOfMemoryError`异常。

## 对象的创建

### 类加载检查

当JVM遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用所代表的类是否已被加载、解析和初始化过。如果没有，则必须先执行相应的类加载过程。

### 新生对象内存分配

在类加载检测通过后，JVM将会为新生对象分配内存。对象所需内存的大小在类加载完成后便可完全确定，为对象分配空间实际上等同于把一块确定大小的内存块从堆中划分出来。分配方式的选择由Java堆是否规整决定，而堆是否规整又由垃圾收集器是否带有空间压缩整理(Compact)的能力决定。

+ 指针碰撞(Bump the Pointer)

假设Java堆中内存是绝对规整的，所有被使用过的内存都被放在一边，空闲的内存被放在另一边，中间是一个指针作为分界点的指示器。此时分配内存就仅仅是把指针向空闲空间方向挪动一段与对象大小相等的距离。

+ 空闲列表(Free List)

如果Java堆中的内存不是规整的，已被使用的内存和空闲内存交错在一起。此时虚拟机必须维护一个列表，记录哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录。

### 对象创建时的线程安全

对象创建在虚拟机中是非常频繁的行为，在并发情况下，可能出现正在给对象A分配内存，指针还没来得及修改，对象B又同时使用了原来的指针来分配内存的行为。解决这个问题有如下两种解决方案：
+ 对分配内存空间的动作进行同步处理，保证更新操作的原子性
+ 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲(ThreadLocal Allocation Buffer, TLAB)。哪个线程需要分配内存，就在哪个线程的TLAB中分配，只有TLAB用完了，分配新的缓存区时才需要同步锁定。

### 对象初始化

(1) 内存分配完成后，JVN将分配到的内存空间（不包括对象头）都初始化为零值。如果使用了TLAB的话，这一项工作也可以提前至TLAB分配时顺便进行。这样做保证了对象的实例字段可以不赋初始值就可以直接使用，此时程序能访问到的是这些字段的数据类型所对应的零值。

(2) 设置对象头。

(3) 执行构造函数，即Class文件中的`<init>()`方法，按照程序员的意愿对对象进行初始化。

## 对象的内存布局

对象在堆中的存储布局可以划分成三个部分：对象头(Header)、实例数据(Instance Data)和对齐填充(Padding)。

### 对象头

包含Mark Word和类型指针两部分：

+ Mark Word，用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。
+ 类型指针，即对象指向它的类型元数据的指针，JVM通过这个指针来确定该对象是哪个类的实例。并不是所有JVM实现都必须在对象头上保留类型指针，当使用句柄访问时便不需要对象头里面有类型指针。
+ 如果对象是一个Java数组，那么在对象头里面还必须有一块用于记录数组长度的数据，以便根据数组长度和数组中对象的元数据信息来推断出数组的大小。

### 实例数据

实例数据部分是对象真正储存的有效信息，无论是从父类继承下来的，还是在这个类中另外定义的字段都必须记录下来。这部分的存储顺序会受到虚拟机分配策略参数和字段在Java源码中定义顺序的影响。默认情况下，相同宽度的字段总是被分配到一起存放，在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。

注意：如果实例字段中有Object类型或者数组类型的数据，存储的实际是指向该Object或数组的指针。

### 对齐填充

任何对象的大小都必须是8字节的整数倍。对象头部分已经被精心设计成8字节的倍数（32位或64位），因此，如果实例数据部分没有对齐的话，就需要通过对齐填充来补全。

## 对象的访问定位

即虚拟机栈上对象的reference数据如何找到堆中具体对象并对其进行操作。主流的访问方式有使用句柄和直接指针两种。

### 句柄访问

Java堆中将会划分出一块内存来作为句柄池。虚拟机栈帧中本地变量表（局部变量表）中的reference实际存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自具体的地址信息。

[![access-by-handler.png](https://i.postimg.cc/8CVzq9Mf/access-by-handler.png)](https://postimg.cc/JsP8XTTR)

使用句柄访问的最大好处是，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中指向对象实例数据的指针，而虚拟机栈栈帧中局部变量表的reference存储的句柄地址则不需要修改。

### 直接指针访问

栈帧局部变量表中reference存储的直接是对象地址，同时对象的对象头中存储了指向对象类型数据的指针。

[![access-by-direct-pointer.png](https://i.postimg.cc/vBbwzd9X/access-by-direct-pointer.png)](https://postimg.cc/KKqWGw0g)

使用直接指针访问最大的好处就是速度快，因为节省了一次指针定位的时间开销。由于对象访问在Java中非常频繁，因此这类开销积少成多也是一项几位可观的执行成本。HotSpot主要使用直接指针进行对象访问。

# 垃圾收集与内存分配策略

## 引用计数算法

引用计数算法会在堆中对象的对象头中存储该对象被引用的次数（不适用于方法区）。每当有一个地方引用该对象时，计数器值加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。例如：
```Java
String m = new String("jack");
```
上述字符串使用了String类的构造方法因此被储存在堆中。字符串`"jack"`被`m`引用，其引用计数为1。

[![set-m-null.png](https://i.postimg.cc/Twtm2fJ4/set-m-null.png)](https://postimg.cc/1nN4JxTG)

```Java
m = null;
```

在将`m`设置为`null`后，字符串`"jack"`的引用计数变为0，因此该字符串将会被被回收。

引用计数算法虽然占用了一些额外的内存空间来进行计数，但其原理简单，判定效率也高。然而，单纯的引用计数很难解决对象之间相互循环引用的问题。

```Java
public class ReferenceCountingGC {
    public Object instance;
    public ReferenceCountingGC(String name) {}
}

public static void testGC() {
    ReferenceCountingGC a = new ReferenceCountingGC("objA");
    ReferenceCountingGC b = new ReferenceCountingGC("objB");

    a.instance = b;
    b.instance = a;

    a = null;
    b = null;
}
```
[![cycle-reference.png](https://i.postimg.cc/cCjsTbdH/cycle-reference.png)](https://postimg.cc/64rJpz2J)

在将`a`和`b`各自的`instance`属性赋值为对方后，`a`和`b`各自的引用计数都变为2，当`a`和`b`被赋值为`null`后，他们各自的引用计数为1。实际上这两个对象已经不可能再被访问，但由于他们的引用计数不为0，因此无法回收他们。

## 可达性分析算法

通过一系列称为GC Roots的根对象作为起始节点集，如果某个对象到GC Roots间没有任何引用链可达，则证明此对象是不可能再被使用的。

[![GC-root-set.png](https://i.postimg.cc/sXgszrXx/GC-root-set.png)](https://postimg.cc/NySW8ZRv)

图中对象object 5, object 6, object 7虽然互有关联，但它们到GC Roots是不可达的，因此它们将会被判定为可回收的对象。

固定可作为GC Roots的对象包括以下几种：

+ 虚拟机栈（栈帧中的本地变量表）中引用的对象
+ 方法区中类静态属性引用的对象
+ 方法区中常量引用的对象
+ 本地方法栈中JNI（即native方法）引用的对象
+ Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象等，还有系统类加载器
+ 所有被同步锁（synchronized关键字）持有的对象
+ 反映Java虚拟机内部情况的JMXBean, JVMTI中注册的回调、本地代码缓存等

基本上，如果把引用分为两种，一种是堆外对象对堆内对象的引用，另一种是堆内对象之间的引用，通常我们所说的GC Roots可以被认为是前者，比如说栈引用堆中对象。由于对象终究是要被外部使用的，比如说被栈引用所访问，因此，如果一大堆的堆内对象之间互相引用，但是没有任何堆外部引用，那么这部分对象实际上也是不可达的。

### 虚拟机栈中引用的对象

```Java
public class StackLocalParam {
    public StackLocalParam(String name) {}
}

public static void testGC() {
    StackLocalParam s = new StackLocalParam("localParam");
    s = null;
}
```
此时，`s`是GC Root，当`s`被赋值`null`时，`localParam`对象与GC Root之间的引用链被断开，因此将会被回收。

### 方法区中类静态属性引用的对象

```Java
public class MethodAreaStaticProperties {
    public static MethodAreaStaticProperties m;
    public MethodAreaStaticProperties(String name) {}
}

public static void testGC() {
    MethodAreaStaticProperties s = new MethodAreaStaticProperties("properties");
    MethodAreaStaticProperties.m = new MethodAreaStaticProperties("param");
    s = null;
}
```
此时，`s`和`m`都是GC Root。当`s`被赋值为`null`后，`properties`对象与GC Root之间的引用链断开，将会被回收。而`param`对象仍与GC Root相连，因此不会被回收。

### 方法区中常量引用的对象

```Java
public class MethodAreaStaticProperties {
    public static final MethodAreaStaticProperties m = MethodAreaStaticProperties("final");
    public MethodAreaStaticProperties(String name) {}
}

public static void testGC() {
    MethodAreaStaticProperties s = new MethodAreaStaticProperties("staticProperties");
    s = null;
}
```
此时，`m`和`s`都是GC Root。`s`被赋值为`null`后，只有`staticProperties`对象对被回收，而`final`对象不会。

## 引用的分类

引用的传统定义： 如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称该reference数据是代表某块内存、某个对象的引用。JDK 1.2后，Java将引用分为以下四种：
+ 强引用(Strongly Reference): 指代码中普遍存在的引用赋值，类似`Object obj = new Object()` 这种引用关系。无论任何情况下，**只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象**。
+ 软引用(Soft Reference): 只被软引用关联着的对象，**在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收**，如果这次回收还没有足够的内存，才会抛出内存溢出异常。
+ 弱引用(Weak Reference): 被弱引用关联的对象只能生存到下一次垃圾收集发生为止。**当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象**。
+ 虚引用(Phantom Reference): 一个对象是否有虚引用的存在，**完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例**。为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知。

## finalize()方法

在可达性分析算法中判定为不可达的对象，离真正被宣告为死亡，还至少要经历两次标记过程：

第一次标记：

如果对象在进行可达性分析后发现没有与GC Roots相连的引用链，那它将会被第一次标记。随后将对其是否有必要执行`finalize()`方法进行筛选。

如果对象没有覆盖`finalize()`方法或其`finalize()`方法已被虚拟机调用过，那么虚拟机将会判断为“没有必要执行”。

如果对象被判定为有必要执行`finalize()`方法，那么该对象将会被放置在一个名为F-Queue的队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的`Finalizer`线程区执行它们的`finalize()`方法。注意，虚拟机虽然会触发这个方法开始运行，但并不承诺一定会等待它运行结束，因为如果某个对象的`finalize()`方法执行缓慢，或者发生了死循环，将很可能导致F-Queue队列中的其他对象永久处于等待，甚至导致整个内存回收子系统的崩溃。

第二次标记：

`finalize()`方法是对象逃脱死亡命运的最后一次机会，稍后收集器将对F-Queue中的对象进行第二次小规模的标记，如果对象要在`finalize()`中成功拯救自己，只要重新与引用链上的任何一个对象建立关联即可，在第二次标记时它将被移除“即将回收”的集合；如果对象这时候还没有逃脱，那基本上它就真的要被回收了。

## 回收方法区

方法区的垃圾收集主要回收两部分内容：废弃的常量和不再使用的类型（指类元信息？）。

### 回收废弃的常量

回收废弃的常量与回收堆中的对象类似。例如一个字符串"Java"曾经进入常量池中，但是当前系统又没有任何一个字符串对象的值是"Java"，换句话说，已经没有任何字符串对象引用常量池中的"Java"常量，且虚拟机中也没有其他地方引用这个字面量。如果在此时发生内存回收，而且垃圾收集器判断确有必要的话，这个"Java"常量就会被系统清理出常量池。常量池中的符号引用也与此类似。

### 回收不再使用的类型

需要同时满足下面三个条件：
+ 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例
+ 加载该类的类加载器已经被回收
+ 该类对应java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

JVN被允许对满足上述三个条件的无用类进行回收，这里说的只是“被允许”，而并不是和对象一样，没有引用了就必然会回收。

## 分代收集理论

三个假说：

1. 弱分代假说(Weak Generation Hypothesis): 绝大多数对象都是朝生夕灭的。
2. 强分代假说(Strong Generation Hypothesis): 熬过越多次垃圾收集过程的对象就越难以消亡。
3. 跨代引用假说(Intergenerational Reference Hypothesis): 跨代引用相对于同代引用来说仅占极少数。如果某个新生代对象存在跨代引用，由于老年代对象难以消亡，该引用会使得新生代对象在收集时同样得以存活，进而在年龄增长之后晋升到老年代中，这时跨代引用也随即被消除了。

前两个假说说明：**收集器应该将Java堆划分出不同的区域，然后将回收对象依据其年龄（即对象熬过垃圾收集过程的次数）分配到不同的区域之中存储**。如果一个区域中大多数对象都是朝生夕灭，难以熬过垃圾收集过程的话，把它们集中放在一起，每次回收时只关注如何保留少量存活而不是去标记那些大量将要被回收的对象，就能以较低代价回收大量的空间；如果剩下的都是难以消亡的对象，那把它们集中放在一块，虚拟机便可以使用较低的频率来回收这个区域。这样做**同时兼顾了垃圾收集的时间开销和内存空间的有效利用**。

使用分代收集理论时，设计者一般至少会把Java堆划分为新生代(Young Generation)和老年代(Old Generation)两个区域。在新生代中，每次垃圾收集时都会有大批对象死去，而每次回收后存活的少量对象，将会逐步晋升到老年代中存放。

在Java堆中划分出不同的区域后，GC才可以每次只回收其中某一个或者某些部分的区域，因而才有了不同回收类型的划分：

**部分收集**(Partial GC): 指收集目标不是完整收集整个Java堆的垃圾收集，其中又分为：
+ 新生代收集(Minor GC / Young GC): 指收集目标只是新生代的垃圾收集。
+ 老年代收集(Major GC / Old GC): 指目标只是老年代的垃圾收集。目前只有CMS收集器会有单独收集老年代的行为。
+ 混合收集(Mixed GC): 指目标是收集整个新生代以及部分老年代的垃圾收集。目前只有G1收集器会有这种行为。

**整堆收集**(Full GC): 收集整个Java堆和方法区的垃圾收集。

分代收集并非只是简单划分一下内存这么容易，它至少存在一个明显的困难，即对象之间不是孤立的，**对象之间存在跨代引用**。假如进行一次只局限于新生代区域内的收集(Minor GC)，在不考虑跨代引用的情形下，一般的做法是找出新生代内与GC Roots相连的对象，在此基础上进行可达性分析，找出该区域中的存活对象。

> When doing minor garbage collection, JVM follows every reference from the live roots to the objects in the young generation, and marks those objects as live, which excludes them from the garbage collection process. (quoted from this [post](https://stackoverflow.com/a/51411524/15595332))

[![intergenerational-reference.png](https://i.postimg.cc/tRZCpPF1/intergenerational-reference.png)](https://postimg.cc/c6WNcv9Z)

上图中红色箭头是堆外对象对堆内对象的引用，红色箭头的起点也就是所谓的GC Roots。如果我们只对新生代中与GC Roots相连的对象进行可达性分析，那么可以断定N, S, P, Q都是存活对象，但是V却不会被认为是存活对象，其占据的内存会被回收。

所以，为了解决这种跨代引用的问题，最笨的办法就是遍历老年代的对象，找出这些跨代引用来。遍历整个老年代所有对象的方案虽然理论上可行，但无疑会为GC带来很大的性能负担。根据第三个假说，跨代引用是极少的，因此可以在新生代上建立一个记忆集(Remembered Set)，来标识出老年代的哪一块内存会存在跨代引用。

[![remembered-set.png](https://i.postimg.cc/wvznt1vd/remembered-set.png)](https://postimg.cc/3yLtVJmL)

如上图所示，记忆集记录了从S到P和从U到V的跨代引用。虚拟机会将记忆集记录的跨代引用（仅指从老年代引用新生代）也当作GC Roots进行处理。虽然这种方法需要在对象改变引用关系时维护记录数据的正确性，会增加一些运行时的开销，但比起扫描整个老年代来说仍是划算的。

### 记忆集与卡表

记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。收集器只需要通过记忆集判断出某一块非收集区域是否存在有指向了收集区域的指针就可以了。记忆集中的记录精度可以设计为字长精度（处理器寻址精度，32位或64位）、对象精度（每个记录精确到一个对象）、卡精度（每个记录精确到一块内存区域）。其中，第三种精度实现是用一种称为“卡表”(Card Table)的方式去实现记忆集。

卡表最简单的形式可以只是一个名为CARD_TABLE的字节数组，其每一个元素都对应着其标识的内存区域中一块特定大小的内存块，这个内存块被称作卡页(Card Page)。

[![card-table-card-page.png](https://i.postimg.cc/DZWL3tmN/card-table-card-page.png)](https://postimg.cc/9RhrGgdP)

[![card-table-card-page2.png](https://i.postimg.cc/LsHPCJzN/card-table-card-page2.png)](https://postimg.cc/hX5hhPZd)

一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或多个）对象的字段存在着跨代指针，那就将对应卡表的数组元素的值标识位1，称为这个元素**变脏**(Dirty)，如果没有则标识为0。在垃圾收集发生时，只要筛选出卡表中变脏的元素，就能轻易得出哪些卡页内存块中包含跨代指针，把它们加入GC Roots中并进行可达性分析。

### 写屏障

**卡表何时变脏**：当老年代中的对象引用了新生代中对象时，老年代对象对应的卡表元素就应该变脏，变脏时间点原则上应该发生在引用类型字段赋值的那一刻。

**如何维护卡表状态**：HotSpot是通过写屏障(Write Barrier)技术维护卡表状态的。赋值的前后都在写屏障的覆盖范围内，一旦JVM遇到这些写屏障，便会对卡表进行更新操作。虽然更新卡表会产生额外的开销，但这个开销与Minor GC时扫描整个老年代的代价相比还是低得多。
> When an object in old generation writes/updates a reference to an object in the young generation, this action goes through something called write barrier. When JVM sees these write barriers, it updates the corresponding entry in the card table. (quoted from this [post](https://stackoverflow.com/a/51411524/15595332))

**False Sharing**: 当多线程修改互相独立的变量时，如果这些变量恰好处于同一个缓存行(Cache Line)，就会彼此影响(写回、无效化或者同步)而导致性能降低。

> The cache records cached memory locations in units of cache lines containing multiple words of memory. A typical cache line might contain 4–32 words of memory. On a cache miss, the cache line is filled from main memory. So a series of memory reads to nearby memory locations are likely to mostly hit in the cache. When there is a cache miss, a whole sequence of memory words is requested from main memory at once. This works well because memory chips are designed to make reading a whole series of contiguous locations cheap. (quoted from this [page](https://www.cs.cornell.edu/courses/cs3110/2012sp/lectures/lec25-locality/lec25.html))
>
> "false sharing" is something that happens in (some) cache systems when two threads (or rather two cores) writes to two different variables that belongs to the same cache line. In such cases the two threads/cores competes to own the cache line (for writing) and consequently, they'll have to refresh the memory and the cache again and again. That's bad for performance. (quoted from this [post](https://stackoverflow.com/a/61632872/15595332))

假设处理器的cache line大小为64 bytes，由于一个卡表元素占1 byte，64个卡表元素将处于同一个cache line中。这64个卡表元素所对应的卡页的总的内存为32 KB (64 $\times$ 512 bytes)，也就是说如果不同线程更新的对象更好处于这32 KB的内存区域内，就会导致更新卡表时正好写入同一个cache line而影响性能。为了避免False Sharing问题，可以在每次JVM遇到写屏障需要更新卡表时，先检查卡表标记，只有当该卡表元素未被标记过时才将其标记为变脏。

## 根节点枚举

可达性分析可分为两个阶段：

1. 根节点枚举，找出所有的GC Roots
2. 从根节点开始查找引用链

目前查找引用链这一过程已经可以做到与用户线程一起并发，但根节点枚举这一过程仍必须暂停用户线程才能进行(Stop the World)。在枚举期间，整个系统看起来像被冻结在某个时间点上，不会出现在分析过程中，用户进程还在运行，根节点集合的对象引用关系还在不断变化这种情况。如果这点都不能满足的话，可达性分析结果的准确性也就无法保证。

固定可作为GC Roots的结点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）中。尽管目标很明确，但把方法区中的常量或类静态属性和栈帧中的本地变量表等区域全部扫描一遍以找出对象引用实在太过于费时。

OopMap: 

[link 1](https://zhuanlan.zhihu.com/p/441867302)

[link 2](http://09itblog.site/?p=901)

[link 3](https://www.pudn.com/news/628f83bdbf399b7f351eb02d.html)


## 标记-清除算法

[![mark-sweep-algorithm.png](https://i.postimg.cc/xCCZxxcw/mark-sweep-algorithm.png)](https://postimg.cc/67kzThVc)

## 标记-复制算法

[![mark-copy-algorithm.png](https://i.postimg.cc/VkkHLZFd/mark-copy-algorithm.png)](https://postimg.cc/qzWG1LST)

## 标记-整理算法

