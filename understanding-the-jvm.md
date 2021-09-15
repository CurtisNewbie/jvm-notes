# 深入理解 Java 虚拟机 (2019) 周志明

JVM Spec: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html

GC Basic: https://codeahoy.com/2017/08/06/basics-of-java-garbage-collection/

GC Basic and implementation: https://plumbr.io/handbook/garbage-collection-algorithms-implementations#g1

# 2. 自动内存管理

## 运行时数据区域

根据 Java 虚拟机规范，虚拟机在运行 Java 程序的时候会将管理的内存栈分为以下五个部分。其中，方法区和堆是所有线程共享的数据区。虚拟-机栈，本地方法栈和程序计数器时线程隔离的数据区.

- 虚拟机栈 VM Stack
- 本地方法栈 Native Stack
- 堆 Heap
- 方法区 Method Area
- 程序计数器 Program Counter Register

## 程序计数器 Program Counter Register

程序计数器时一块较小的内存空间，用于记录的当前线程正在执行的字节码指令的地质，相应的字节码解释器工作时，就是通过改变这个计数器的值-来选取下一条需要执行的字节码指令。所以程序控制流，如，分支，循环，跳转，异常处理等，都依赖于程序计数器来完成。程序计数器是线程隔离-的，每条线程都有独立的计数器.

## Java 虚拟机栈 VM Stack

JVM Stack 是线程私有的，它的生命周期与线程相同。Jvm Stack 用于描述 Java 方法执行的线程内存模型; 每个方法被执行的时候，JVM 都-会创建一个栈帧 (Stack Frame) 用于存储局部变量表，操作数栈，动态连接，方法出口等信息.

1. 方法调用 
    - Stack Frame 创建且入栈 
2. 方法执行完毕 
    - Stack Frame 出栈

### 局部变量表

局部变量表中存放了 compile-time 可知的基本数据类型，对象引用，和 returnAddress (指向字节码指令地址)。这些信息在局部变量表中以-slot (局部变量槽) 的形式表示。一个 slot 的大小由具体实现的 JVM 决定，一般 64 位的 long 和 double 使用两个 slot。一个 Stack Frame 的局部变量表锁需要的 slot 的数量在编译时时完全可以确定的，所以在方法运行时，局部变量表大小不变。当出现无法分配足够内存给 Stack Frame 时，JVM 抛出 OOM 异常.

- 以 slot 局部变量槽为单位
- 数据类型
- 对象引用
- returnAddress (字节码地址)

## Java 堆 Heap

Java Heap 是所有内存共享的，在 JVM 启动时创建。此数据区域唯一目的为存放对象实例。因为 Java heap 存储对象实例，此区域也是 GC 管理的区域，时称 GC Heap。参数 -Xmx 和 -Xms 设定的即是 Heap 的大小。当 Heap 没有足够内存进行分配时，JVM 会抛出 OOM 异常.

## 方法区 Method Area

(p.46)

方法区是线程共享的内存区域，用于存储已被虚拟机加载的类型信息，常量，静态变量，编译后代码等信息。当 JVM 启动时，JVM 会创建 Method Area，为了与 Heap 区分开，它也被称为 Non-Heap。当方法区没有足够内存用于分配的时候，OOM 错误被抛出.

### 运行时常量池 Runtime Constant Pool

根据 JVM Spec: 

***"A run-time constant pool is a per-class or per-interface run-time representation of the constant_pool table in a class file. It contains several kinds of constants，ranging from numeric literals known at compile-time to method and field references that must be resolved at run-time."***

运行时常量池是方法区的一部分。Class 文件中除了有类的版本，字段，方法，接口等信息外，还有一个常量池表 (Constant Pool Table)，用-于存放各种编译期间生成的字面量和符号引用。由于类是可以动态加载的，加载时，类中的常量池表也会一同动态加载到运行时常量池中。当不足以-分配内存时，抛出 OOM 错误.

注: 符号引用 = Symbolic Reference

## 本地方法栈 Native Method Stack

与 VM Stack 工作方式相同，Native Method Stack 用于 VM 调用本地方法，同样的当没有内存的时候，抛出 OOM 错误.

## HotSpot VM 对象

(p.49)

### 1. 对象创建

对象创建使用 `new` 指令。当 JVM 遇到 `new` 指令的时候，它会做以下事情:

1. 检查该指令参数是否能在运行时常量池 (Runtime Constant Pool) 中定位到该类的符号引用 (Symbolic Reference)，同时检查该类是-否已加载和解析，如果没有，则加载该类到方法区中，包括常量池。

2. 当类加载通过时，JVM 为对象分配内存。同样，当类被加载时，一个对象所需要的大小是已知的。JVM 从 Heap 中分配内存即是为对象分配空-间的做法。

    - 当 JVM Heap 中内存为绝对规整时（所有使用中的内存在一边，未使用的内存在另一边，中间为分配内存的指针），则分配 Heap 内存就-只是简单的移动中间的指针，使得分配的内存大小与对象所需大小相同。这种方法较 '指针碰撞' (**Bump the Pointer**)

    - 当 JVM Heap 中内存并不规整时, JVM 则必须维护并记录哪些内存时可用的或已分配的。这种方式称为 '空闲列表' (**Free List**)

    - JVM Heap 中内存是否规整取决于 GC, 带压缩整理能力的 GC 可使用 **Bump the Pointer** 的方式，但当使用 **CMS Concurrent Mark Sweep** 类型的 GC 时，则只能使用较复杂的方式进行分配。

    - 为了更快的内存分配，CMS 类型 GC 使用 TLAB 解决方案。**TLAB Thread Local Allocation Buffer** 是 Heap 为每个线程预-分配的，可看作是线程的本地缓冲区，只有在 TLAB 用完，而且由 Heap 分配新的 TLAB 时才需要进行同步锁定，不过当 TLAB 内部分配-内存存在并发的时候，也需要使用锁。每个 TLAB 内部也可以使用不同的内存分配策略。

3. 当对象空间分配结束后，JVM 需要对对象进行设置。如，对象所属的类，如何找到该类的元数据，对象 GC 分代信息等存储在 **Object Header** 中的信息。

4. 初始化对象的值

## 对象的内存布局

(p.51)

在 HotSpot JVM 中，对象在 Heap 中存储分为三个部分:

1. 对象头 Object Header
2. 实例数据 Instance Data
3. 对齐填充 Padding

### 对象头

对象头包括两类信息:

1. 对象自身的运行数据, 不是实例数据 **Mark Word**

    - 哈希码
    - GC 分带年龄
    - 锁状态标记
    - 线程持有的锁
    - 便向线程的 ID 
    - 偏向时间戳

2. 对象类型元数据的指针 **Klass Word**

    - JVM 通过类型元数据的指针来确定该对象是哪个类的实例。如果对象是一个数组，那在对象头中还需要记录数组的长度。


### 对齐填充

(p.52)

Padding 没有特别的含义，只是起到填充的作用。由于 HotSpot JVM 要求对象的大小必须为 8字节的整数倍，当对象实例数据没有对齐时，则需-要 Padding 来对齐。

## 对象的访问定位

(p.52)

程序通过 **VM Stack** 虚拟机栈上的 reference 引用对象，对象的访问方式有两种:

1. 通过句柄来访问

    - 句柄为 handle, 也就是 reference of external resource

2. 直接使用指针访问

### 通过句柄

JVM 在堆 Heap 中划出一部分内存作为句柄池 (pool of handle)，句柄是指 "reference to a resource that is managed externally"。可以简单的看作是指针的地址。Heap 中分隔为句柄池和实例池，访问对象的方式如下:

```
[Stack]     [Heap]

ref --------> handle  -----------> instance data
              |
              |
            [Method Area]
              |
              |
              v
            Class & Metadata
```

### 通过指针 

```
[Stack]     [Heap]

ref --------> instance data 
              |
              |
            [Method Area]
              |
              |
              v
            Class & Metadata
```

### 指针和句柄

直接使用指针访问更快，而使用句柄增加了一层 abstraction，当对象被移动时（因为GC），这只会改变句柄的地址而不是 stack 中的 reference。

# 3. 垃圾收集器与内存分配策略

## 判断对象是否仍存活

### 引用计数算法 Reference Counting

此做法为，在对象中加一个引用计数器，当有一个地方引用它，计数加一，当有一引用失效，计数减一。此算法原理简单，在大多数情况下效率较高。-但当遇到例外情况时，则需要大量的额外处理才能正常工作。例如，当两个对象相互引用时，他们的计数器都加1，即使他们不会在后续代码中访-问，他们也无法被简单的回收。

### 可达性分析算法 Reachability Analysis

Reachability Analysis 是目前主流使用的算法。基本思路是，使用被称作 **GC Roots** 的根对象作为起始的节点，以树 traversal 的-方式向下通过引用关系搜索，其中的 path 即是 **Reference Chain 引用链**。如果某一对象与 **GC Roots** 之间没有任何 **Reference Chain**， 则证明该对象不可达，也就是可回收。程序中会维护多个 **GC Roots**，也可看作 **GC Root Set**. Effectively, **ClassLoader** 就是一个 GC Root.

### 引用

在 Java 中，引用可分为以下四种，依次逐渐减弱：

1. 强引用 Strong Reference
    - 作为最传统的引用定义，无论何种情况，只要存在强引用，对象都不会被 GC
2. 软引用 Soft Reference
    - 用于描述有用，但非必须的对象，但内存将要溢出的时，JVM 将软引用标记为回收范围内
3. 弱引用 Weak Reference
    - 用于描述非必须对象，不管是否发现内存不足，弱引用对象都会被回收
4. 虚引用 Phantom Reference
    - 也被称为 '幽灵引用' 或 '幻影引用'，使用它的唯一目的是该对象被回收时可收到系统通知

### 标记回收后的对象

当对象被可达性分析标记为不可达时，该对象也不会立刻被回收。一个对象至少要被标记两次才会被真正地回收。当一个对象第一次被标记为不可达-时，JVM 检查该对象是否需要执行 `Object.finalize()` 且是否已经执行过该方法，该方法只能被执行一次。如果需要执行，则把该对象方知道-名为 `F_QUEUE` 的队列中，等待执行。JVM 只负责触发 `F_QUEUE` 中对象的 `finalize()` 方法，但不会等待他们结束。同时，当该对象-再次被 GC 标记，该对象会被回收。

### 回收方法区

方法区的回收，如，类卸载，所能带来的收益往往较低。JVM 规范不要求 JVM 支持类卸载等方法区回收的功能，对方法区内数据的回收往往基于苛-刻的判定条件，但当存在大量使用反射，动态代理，CGlib 等字节码框架时，一般需要 JVM 具备方法区回收的能力，以确保不会对方法区带来过大-压力。

## 垃圾收集算法

垃圾收集算法基本可分为:

1. Reference Counting GC 引用计数式垃圾收集
2. Tracing GC 追踪式垃圾收集

### 分代收集理论

多数 GC 遵循了 **Generational Collection 分代收集理论**, 该理论基于两个假说之上:

1. 弱分代假说 Weak Generational Hypothesis
    - 绝大多数对象都是朝生夕灭的
2. 强分带假说 Strong Generational Hypothesis 
    - 通过越多次 GC 的对象越难于消亡

基于以上理论和假设，收集器将 Heap 划分为不同的区域，将对象依据年龄 (通过 GC 的次数) 分配在不同的区域进行存储。由此，将年龄较大的-放同一区域，该区区域则不需要进行频繁的 GC。相反，年龄较小的对象，GC 与其标记大量消亡的对象，不如只关注标记少量存活的对象。

当 Heap 划分为不同分代区域时，GC可每次只进行部分 GC (Partial GC): 

- Minor GC / Young Gc 新生代 GC
- Major GC / Old GC  老年代 GC
- Mixed GC  混合 GC
- Partial GC 部分 GC
- Full GC 完全 GC

通过将 Heap 划分为两个或以上的分代区域，不同的 GC 算法可应用到不同的区域中，如：

- 标记-清除算法 Mark-and-Sweep
- 标记-复制算法 Mark-and-Copy
- 标记-整理算法 Mark-and-Compact

但实际应用中会出现跨代引用的情况，对应这个问题则有第三条假设:

3. 跨代引用假说 Inter-generational Reference Hypothesis
    - 跨代引用相对于同代引用只占极少数

根据该假说，当扫描新生代时，不必为了跨代引用去扫描老年代，只需要在新生代上建立全局的数据结构 **记忆集 Remember Set**， 用于将老-年代划分为若干个小块，标识出老年代中哪一块会存在跨代引用。当发生 Minor GC (新生代GC) 时，只有被记忆集标记的会存在跨代引用的区域会-加入到 Minor GC 的 CG Roots 中进行扫描。

### 标记-清除算法 Mark-and-Sweep

Mark-Sweep 算法分为两个部分:

1. 标记所有需要回收的对象
2. 统一回收被标记的对象，或反过来，标记存活对象。

该算法最早出现，也是最基础的算法，但是存在两个根本问题:

1.  执行效率不稳定
    - 大量的对象意味着大量的标记和清除，会带来性能问题
2.  标记清除后带来的内存随便化问题
    - 内存碎片化会降低对象空间分配的效率，若有较大的对象，可能会出现无法找到足够大的连续内存的情况

### 标记-复制算法  Mark-and-Copy

该算法将内存分为两个部分，每次新分代分配内存只会使用其中的一部分，与其使用标记清除的方式，该算法直接将该部分中存活的对象复制到另一部-分未在使用的内存中，这样内存是连续的，且无需要考虑内存碎片化的问题，这做法也是基于 "大部分新生代对象都无法活过 GC" 的假设。

具体例子，在 HotSpot 中，新生代称为 Young Generation，其中包含 **Eden** 和 **Survivor**， Eden 与 Survivor 比例为 8:1，也就是有一个 Eden 和两个 Survivor。新对象会在 Eden 中分配，当 Eden 满了的时候，发生 Minor GC，新生代中存活的对象会被复制到 Survivor 中。当 Survivor 也满了的时候, 为了尽可能避免 Major GC 和使用过多 Old Generation 的内存。此时，已经使用了的那 10% Survivor 中存活的对象则被复制到另一个 **未被使用** 的 10% Survivor 中。这样能确保，不管是 Eden 还是 Survivor 内存都是连续的，避免碎片化。但是这样也有一个问题，那就是总是有一个 Survivor 未被使用。

```
Young Generation: 

1 x Eden (80%)
2 x Survivor (20%)
```

### 标记-整理算法 Mark-and-Compact

标记和复制算法在对象存活率比较高的情况下，就会出现较多的复制操作。同时，为了应对100%对象都在 GC 存活的情况下，该算法一般不用于老年-代而是新生代。标记-整理算法较适用于老年代。与标记清除算法相似，该算法分为两个部分:

1. 标记存活对象
2. 统一移动对象位置

因为内存被读取和分配的频率相比整理来说多得多，所以该算法是有其优点的。

# 7. 虚拟机类加载机制

**JVM 类加载机制**:

JVM 把描述类的数据从 class 文件 (byte stream) 加载到内存中，并且对数据进行校验，转换解析和初始化，到最后成为可以直接被 JVM 使-用 Java 类型的完整过程。

## 类加载的时机

一个类型 (class / interface) 从被加载到 JVM 内存中到卸载出内存，整个生命周期包含七个阶段:

1. 加载 Loading
2. 验证 Verification
3. 准备 Preparation
4. 解析 Resolution
5. 初始化 Initialization
5. 使用 Using
7. 卸载 Unloading

其中，2-4 被称为 连接阶段 Linking.

加载，验证，准备，初始化，和卸载这五个阶段的顺序是确定的。但解析阶段在某些情况下可以在初始化之后进行（为了动态绑定，晚期绑定）。同-时，这些阶段是按顺序开始的，但可能交叉的混合进行。

**六种情况必须对类初始化**:

JVM 规范对初始化的实际有严格的要求，只有以下的六种情况必须立即对类进行初始化（包括，加载，验证，准备），如果该类型仍未进行初始化。

1. 遇到 new, getstatic, putstatic, invokestatic 这四种 bytecode 时，如:
    - 使用 new 实例化对象
    - 读取或设置一个 static non-final 的字段
    - 调用一个类型的静态方法

2. 使用 java.lang.reflect 对类型进行反射调用
3. 当初始化子类时，父类如果没有初始化，也需要初始化
4. 包含 main() 方法的类
5. 当使用动态语言支持, java.lang.invoke.MethodHandle 且解析结果未 REF_getStatic, REF_putStatic, REF_invokeStatic, REF_newInvokeSpecial 四种句柄(handle)时, 对应的类需要初始化
6. 带有 default method 的 interface 的实现类进行了初始化时

## 类加载的过程

### 加载 Loading

Loading 阶段主要负责完成三件事:

1. 通过类的全限定名来获取该类的二进制字节流
2. 将该字节流所代表的静态存储结构转化未方法区中的运行时数据结构
3. 在内存中 (Heap 中) 生成一个代表这个类的 `java.lang.Class` 对象
    - 注意, 关于这个类的信息仍然储存在方法区中

### 验证 Verification

验证是连接 Linking 的第一个阶段，这一阶段确保 class 文件字节流符合 JVM 规范要求，同时防止恶意字节流码进入 JVM 中，这个阶段包括-四个步骤:

1. 文件格式验证
    - 未进入方法区
    - 关注 class 文件中的字节流是否符合要求，这决定着这段字节流是否能被转化未方法区内的数据格式
2. 元数据验证
    - 未进入方法区
    - 关注类的语义分析，即该类是否继承了错误的类等
3. 字节码验证
    - 进入方法区内
    - 通过数据流的分析和控制刘的分析，检查语义是否正确，同时确保类的方法在运行时不会做出不允许的行为
4. 符号引用验证
    - 进入方法区内
    - 发生在 symbolic reference 转化为直接引用时，这是确保外部类时存在的且依赖是可以使用的

### 准备 Preparation

该阶段负责将类中定义的静态变量分配内存并设置默认值, 如, int 类型分配 0，long 类型分配 0L。注意，即使代码中对于 non-final 的静-态设有值，这一阶段并不会直接设代码中的值，而是设置默认值，除非该变量是 final 的，具体如下:

```
private static int num = 123;
```

上面这个例子，在 Preparation 阶段，设置的值仍然是 0，因为它不是 final 的。值 123 是在初始化阶段分配的，而不是准备阶段。同时-这些属于类的静态变量会与创建的 Class 对象一同放置在 Java Heap 中。如果静态变量是不可变的，则该阶段会直接设置被赋予的值，具体如下

```
private static final int num = 123;
```

因为该静态变量是 final，在准备阶段，num 就已经被设置为 123。

### 解析 Resolution

解析是将 **Symbolic Reference** 转换为 **Direct Reference** 的过程。同时，解析阶段也会检查方法/字段中的 **Access Specifier**， 更具体的说，该阶段解析的有:

- 类
- 接口
- 字段
- 类方法
- 接口方法
- 方法类型
- 方法句柄 
- 调用点限定符 Dynamically-Computed Call Site Specifier

### 初始化 Initialization

类的初始化阶段是类加载的最后一个步骤。直到初始化阶段，JVM 才真正开始执行类中编写的 Java 代码，初始化的有

1. 类的 non-final 变量
2. 类的 static 代码

## 类加载器 ClassLoader

ClassLoader 负责类加载中的 Loading 阶段，更具体的是 "通过一个类的全限定名来获取描述该类的二进制字节流“ 的动作。类的加载实际上就是以下-两个元素的加载:

- 类的全限定名
- 类的二进制字节流

## 三层类加载器

Java 使用三层类加载器结构，具体如下:

1. 启动类加载器 Bootstrap ClassLoader

    - 负责加载 `<JAVA_HOME>/lib`, 一般有 C++/C 实现，作为 JVM 的一部分

2. 扩展类加载器 Extension ClassLoader

    - 负责加载 `<JAVA_HOME>/lib/ext`, 作为 JDK 扩展的加载 (使用 Java)

3. 应用类加载器 Application ClassLoader

    - 也称作系统加载器，负责加载用户路径 (classpath) 所有的类，一般作为默认的 ClassLoader

如果没有写自定义的加载器，一般使用的都是 Application ClassLoader / System ClassLoader。当需要编写自定义 ClassLoader，通常也是通过扩展-该 ClassLoader。一般扩展的目的有，如增加出磁盘以外的 class 存储方式，或，将 class文件二进制字节流存储在数据库中，或，通过自定义 ClassLoader 实现类的隔离和重载等。

除去前面说的特殊情况下，ClassLoader 一般使用 **Parent Delegation Model 双亲委派模型** 进行实现。这指的是，每个 ClassLoader 以 composition 的方式包含它的 parent ClassLoader， 当类加载收到加载类的请求时，它会先使用 parent ClassLoader 进行加载，只有当父级没有加载到该类-的时候，该加载器才会自己加载，在三层类加载器结构也是如此，优先 Bootstrap ClassLoader 进行加载，失败的话 Extension ClassLoader 加载，如果仍然失败，Application/System ClassLoader 加载。

这样设计，无论哪个加载器进行加载，都会最终委派到模型最顶端的启动类加载器，确保公用的类都是由同一个加载器加载出来的。由于 ClassLoader 维持着层级关系，高层级的类 (如，由Bootstrap ClassLoader加载的) 能确保大家使用的是'同一个类'。

对于 JVM 来说，一个独一无二的类由两个因素来确定:

- 加载此类的 ClassLoader 
- 此类的全限定名

也就是说，同一个 ClassLoader 加载出来的一个类的全限定名只有一个，但是一样的全限定名的类但是由两个不同的 ClassLoader 加载出来，在 JVM -中是不相同的 (如，使用 equals 方法会返回 false)。这种可以看成不同 ClassLoader 带来的类的不同和隔离。这样也有助于实现如，webapp 之间的隔离，不同版本依赖的隔离。

# 12. Java 内存模型与线程

## 主内存与工作内存

JMM 内存模型规定，所有变量 (Heap 中的) 必须存储到主内存 (Main Memory) 中，同时每条线程还有自己的工作内存 (Working Memory)。线程对变量的-所有操作 (读/写) 都必须在工作内存中完成，而不能直接读写主内存中的数据。线程之间变量值的船体必须通过主内存来完成。工作内存并不真的存-在，但我们可以把它大致的看作是栈中变量从主内存拷贝回来的值，或者说是寄存器中的缓存。

```
Java Thread <-> Working Memory <-> Read/Write <-> Main Memory
```

## 主内存与工作内存交互工作

关于主内存和工作内存之间的交互，JMM 规定了8种操作，同时要求这8种操作必须是原子的:

1. lock 

    - 作用于主内存，标记变量被某一线程独占

2. unlock

    - 作用于主内存，释放变量锁定状态

3. read

    - 作用于主内存，将变量值从主内存传到工作内存中

4. load

    - 作用于工作内存，将从主内存传输的变量值，放入工作内存的副本中

5. use

    - 作用于工作内存，将工作内存中变量值传递给执行引擎 

6. assign

    - 作用于工作内存，将从执行引擎接受的值赋给工作内存中变量

7. store

    - 作用于工作内存，将工作内存某一变量值传输给主内存

8. write

    - 作用于主内存，将传输给主内存的值写如主内存变量中

### 从 Main Memory 传输给 Working Memory

`read` 和 `load` 必须成双出现，按顺序。

```
load 2)
  | 
  | 
  v                    read 1) 
working memory <------------------ main memory
```

### 从 Working Memory 传输给 Main Memory

`store` 和 `write` 必须成双出现，按顺序。

```
                                      write 2)
                                        |
                                        |
                       store 1)         v 
working memory -------------------> main memory
```

### 使用执行引擎

`use` 和 `assign`

```
                   use
working memory ------------------->
                                     engine
working memory <-------------------
                   assign
```

### 8种操作和对应的规则

1. read/load 和 store/write 必须成双出现，但不一定要求连续执行，也就是说中途可以有其他的指令执行
2. 不允许线程丢弃最近的 assign, 也就是说该 use/assign 的结果必须同步到主内存中
3. 不允许线程无原因地 (没有发生过 use/assign) 就同步数据到主内存中
4. 新的变量只能在主内存中诞生，也就是不允许在工作内存中直接使用未初始化变量，对一变量使用use/store之前，必须使用assign/load，也-就是必须先从主内存同步到工作内存中
5. 对一变量使用 lock 会清除该变量工作内存的值，使用执行引擎 use/assign 之前必须使用 read/load 赋值，这样确保 lock 带来 visibility
6. 只有拥有 lock 的线程可以 unlock
7. 在对一变量进行 unlock 前，必须将变量同步回主内存中，也就是进行 store/write，这样确保下一线程对该更改可见

## 对于 Volatile 变量的特殊规则

当一变量被定义为 volatile 时，它具备两种特性:

1. 保证该变量对所有线程的可见性

    - 当某一线程修改了该变量的值，改变的值对其他所有线程立刻可见 (如，限制工作内存的缓存-来保证可见性) 但可见性不代表是线程安全的，只有当满足以下两个条件才可以认为 volatile 的使用是线程安全的。

        1. 运算结果不依赖当前值，或确保只有单一线程修改变量
        2. 变量不需要与其他变量共同参与不变约束 

2. 禁止指令重排序优化
    - 这里指的是代码必须按编写的顺序执行，不能用重排序优化代码。重排序是一种编译器，运行时和硬件的优化，重排序可在保证结果不被影-响的情况下进行一些优化, 这也被称为 "Within-Thread-As-If-Serial-Semantics", 但是在多线程的情况下，这样的优化可能有意外的结-果，所以需要适当的控制。

对于 volatile 和内存操作结合的一些规则:

如, 有线程 T, volatile 变量 v 和 w

1. T 对 v 的读取和使用，必须满足连续的 1) read, 2) load, 3) use, 对于 T 来说，这些动作必须连续且一起出现，这样才能确保在对 volatile 变量的可见性 

2. T 对 v 的写，必须满足连续的 1) assign, 2) store, 3) write, 对于 T 来说，这些动作必须连续且一起出现，这样才能保证对 volatile 更改能让其他线程立刻可见


3. 假设 T 同时对 v 和 w 进行操作则，如果 A 早于 B 则有 P 早于 Q

```
T -> v:

A      F        P
use    <- load  <- read 
assign -> store -> write 

T -> w:

B      G        Q
use    <- load  <- read 
assign -> store -> write 
```

## 对 long 和 double 型 64 位变量的特殊规则

尽管 JMM 对 lock, unlock, read, load, store, write, use, assign 都要求原子性，但内存模型允许 long 和 double 类型分两次32位操作，即 Non-Atomic Treatment of double and long variables。经过实验，主流 JVM 都保证了原子性操作，所以不需要特意关注此规则，注意，volatile 可强制对 long 和 double 的读写改为原子操作。

## 原子性，可见性与有序性

- 原子性 Atomicity
    
    - 保证8种操作为原子性

- 可见性 Visibility

    - 指当一个线程修改了某一变量时，其他线程可立刻得知这个修改，volatile 保证新值的更改立即同步到主内存中，unlock 要求在解锁-前将更改的值同步到主内存中

- 有序性 Ordering
    
    - 对于单线程保证 Within-Thread-As-If-Serial Semantics

## 先行发生原则

Java 语言存在先行发生原则 Happens-Before，可用于判断数据是否存在竞争，是否安全等。先行发生是 JMM 中定义的两项操作之间的偏序关系, 如果-操作A先行发生在操作B之前，操作A产生的影响能被操作B观察到。

JMM 中天然的先行关系

1. Program Order Rule 程序次序规则
    
    - 同一线程内，控制流顺序确保写在前面的操作先行发生于后面的操作

2. Monitor Lock Rule 管程锁定规则

    - 一个unlock 操作先行发生于后面对同一个锁的 lock 操作

3. Volatile Variable Rule Volatile 变量规则

    - 对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作和写操作

4. Thread Start Rule 线程启动规则

    - Thread.start() 先行发生于该线程的每一个动作

5. Thread Termination Rule 线程终止规则

    - 线程中所有操作先行发生于对此线程的终止检测

6. Finalizer Rule 对象终结规则

    - 一个对象的初始化完成，先行发生于它的 finalize()

7. Transitivity 传递性

    - 如果操作A先行发生于B，B先行发生于C，则有A先行发生于C

先行关系确保，当操作 A 先行发生于操作 B，A的影响对B可见，无须任何同步手段，以上规则确保先行关系，JVM也会保留以上关系，即使存在排序-优化。先行关系不指时间上的先后，而是一个 **"guarantee of ordering of read and write to memory"**, 可以防止 **memory inconsistency errors**.
    

## Java 与线程

实现线程的方式有三种

1. 使用内核 1:1 

    - 内核线程 KLT Kernel-Level Thread 是直接由 Kernel 支持的，通过 Kernel 完成线程切换，使用 Scheduler 调度器进行调度。程序一般-不直接使用 KLT，而内核提供一种轻量级进程 Light Weight Process LWP, 每个 LWP 都由一个 KLT 支持，但我们一般指的线程都是 LWP。一个进程可以有 N 个 LWPs, LWP 与 KLT 1:1 的关系。

    - 由于每个 LWP 都有 KLT 支持，对线程的操作都需要进行系统调用。系统调用是通过内核的 System Call Interface，当进行 SCI 调用，进-行需要切换用户态 User Mode 到内核态 Kernel Mode，这是硬件层面的控制，代价也很大。

2. 使用用户线程 1:N
    
    - 用户线程 User Thread, 广义的来说，只要不是 KLT 都是 UT 的一种。UTs 对于内核来说应该是不可感知的，UT 的建立，同步，销毁和调-度完全在用户态完成，不需要内核支持，用于高并发。

3. 使用用户线程加轻量级进程混合 N:M

    - 混合实现，将内核线程和用户线程一起使用，许多 UNIX 系统都支持 M:N 实现。

## Java 线程调度

调度的方式主要有两种:

- 协同式 Cooperative Thread Scheduling
- 抢占式 Preemptive Thread Scheduling

协同式由线程自行控制调度，当某一线程完成后，自行同志系统切换线程。抢占式是由系统进行调度，线程的执行时间由系统控制，同时，程序是允许提-供建议给调度器的，如，设置线程优先级。

## 线程状态切换

Java 中定义了6种线程的状态，在任意时间点，某一线程只能有一种状态

1. New 

    - 创建后尚未启动

2. Running (Runnable)

    - 正在运行或准备运行
    - `Thread.start()`
    - `Thread.notify()`
    - `Thread.notifyAll()`

3. Waiting
    
    - 无限期等待, 不分配 CPU 时间
    - `Thread.wait()`

4. Timed Waiting

    - 有限期等待，不分配 CPU 时间
    - `Thread.sleep()`

5. Blocked 

    - 阻塞中，例如等待排他锁
    - synchronized
    - tryLock

6. Terminated

    - 终止
    - `Thread.run()`

# 补充 GC Algorithm Implementation 

GC Algorithm Implementation: https://plumbr.io/handbook/garbage-collection-algorithms-implementations

- Serial GC

    - Mark-Copy for young generation
    - Mark-Sweep-Compact for old generation 
    - Single thread for GC

- Parallel GC

    - Mark-Copy for young generation
    - Mark-Sweep-Compact for old generation 
    - Multiple thread for GC

- Concurrent Mark and Sweep

    - Mark-Copy for young generation
    - Mark-Sweep for old generation 
    - Multiple thread for GC
    - Doesn't stop the world during all phases, but do compete CPU time with the application threads
    - Problems:
        - Fragmentation for full GC
        - Lack of predictability
    - Full GC
        - Phase 1: Initial Mark
            - This phase will stop the world
            - The goal of this phase is to mark all the objects in the Old Generation that are either direct GC roots or are referenced from some live object in the Young Generation.

        - Phase 2: Concurrent Mark
            - This phase doesn't stop the world
            - During this phase the Garbage Collector traverses the Old Generation and marks all live objects, starting from the roots found in the previous phase of “Initial Mark”. 

        - Phase 3: Concurrent Preclean
            - This phase doesn't stop the world
            - While the previous phase was running concurrently with the application, some references were changed. Whenever that happens, the JVM marks the area of the heap (called “Card”) that contains the mutated object as “dirty” (this is known as Card Marking).

        - Phase 4: Concurrent Abortable Preclean
            - This phase doesn't stop the world
            - This one attempts to take as much work off the shoulders of the stop-the-world Final Remark as possible. The exact duration of this phase depends on a number of factors, since it iterates doing the same thing until one of the abortion conditions (such as the number of iterations, amount of useful work done, elapsed wall clock time, etc) is met.
            - Think of it as doing some possible marking (some might just require a full-stop) for the "Final Remark" phase to some limited extent.

        - Phase 5: Final Remark
            - This phase will stop the world 
            - The goal of this stop-the-world phase is to finalize marking all live objects in the Old Generation

        - Phase 6: Concurrent Sweep
            - This phase doesn't stop the world
            - The purpose of the phase is to remove unused objects and to reclaim the space occupied by them for future use.

        - Phase 7: Concurrent Reset
            - This phase doesn't stop the world
            - Resetting inner data structures of the CMS algorithm and preparing them for the next cycle

- G1 Garbage First

    - Idea built on top of CMS
    - Multiple thread for GC
    - Doesn't stop the world during all phases, but do compete CPU time with the application threads
    - More predictable and configurable
    - Heap is split into a number of (typically about 2048) smaller heap regions for objects. Each region may be an Eden region, a Survivor region or an Old region. The logical union of all Eden and Survivor regions is the Young Generation, and all the Old regions put together is the Old Generation. This allows GC to incrementally collect the heap, only a subset of the regions, called **Collection Set** is handled at a time, all the young generation regions are included as well as a subset of old regions.
    - Another novelty of G1 is that during the concurrent phase it estimates the amount of live data that each region contains. This is used in building the collection set: the regions that contain the most garbage are collected first. Hence the name: garbage-first collection.
    - Evacuation Phase
        - Fully Young   
            - In the beginning of the application’s lifecycle, G1 initially functions in the fully-young mode. When the Young Generation fills up, the application threads are stopped, and the live data inside the Young regions is copied to Survivor regions, or any free regions that thereby become Survivor.
            - Use the Concurrent Marking described below
        - Mixed
            - Takes advantage of Remember Set to handle cross generation references, or more specifically, cross regions. Each region has a remembered set that lists references pointing to this region from the outside. These references will be regarded as additional GC roots, however, if these references are marked to be garbage, they are ignored.
            - Use algorithm similar to Concurrent Marking described below
    - Concurrent Marking
        - Phase 1: Initial Mark
            - Stops the world
            - This phase marks all the objects directly reachable from the GC roots. 
        - Phase 2: Root Region Scan
            -  This phase marks all the live objects reachable from the so-called root region.
        - Phase 3: Concurrent Mark 
            - This phase is very much similar to that of CMS: it simply walks the object graph and marks the visited objects in a special bitmap. 
        - Phase 4: Remark 
            - Stops the world
            - Like previously seen in CMS, completes the marking process. For G1, it briefly stops the application threads to stop the inflow of the concurrent update logs and processes the little amount of them that is left over, and marks whatever still-unmarked objects that were live when the concurrent marking cycle was initiated.
        - Phase 5: Cleanup
            - Involves short period that stops the world
            - This final phase prepares the ground for the upcoming evacuation phase, counting all the live objects in the heap regions, and sorting these regions by expected GC efficiency, and reclaim empty regions. 



