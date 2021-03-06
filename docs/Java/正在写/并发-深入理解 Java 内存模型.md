# JVM 内存结构&Java 内存模型&Java 对象模型

这三个是截然不同的概念。

- JVM 内存结构：与 Java 虚拟机的运行时虚拟机相关。
- Java 内存模型：与 Java 并发编程相关。
- Java 对象模型：与 Java 对象在虚拟机中的表现形式有关。

# 硬件内存模型

硬件设计者为了解决某些问题而引入一些部件，而这些部件自身的引入也会产生新问题，为了解决这些新问题，硬件设计者又引入其他部件，由此就产生了硬件内存模型来解决这些问题。

## 高速缓存

当程序加载时，指令复制到主内存，当处理器运行程序时，指令又从主内存复制到处理器，这些类似的复制操作减慢了程序的的速度。又由于处理器与主内存之间的差距过大，因此硬件设计者采用了更小更快的存储设备，称为**高速缓存存储器**（cache memory, 简称为 cache 或高速缓存）暂时存放处理器近期可能会需要的信息。

高速缓存是一种存取速率远比主内存大而容量远比主内存小的存储部件，每个处理器都有其高速缓存。高速缓存通常也分为一级缓存、二级缓存、三级缓存，设备访问速度依次降低，容量越来越大，每字节的造价也越来越便宜。引入高速缓存后，处理器在执行读、写内存操作的时候，就可以通过高速缓存进行访问。

高速缓存中存在缓存行，用于存储从内存中读取或者准备写往内存的数据，当处理器根据内存地址在高速缓存的缓存行中找到相应的数据时，那就该内存操作就为缓存命中，否则，该操作就为缓存不命中。缓存不命中也会影响处理器的执行性能。

## 缓存一致性协议

当多个线程访问同一个共享变量时，这些线程执行处理器的高速缓存中都各自保存一份该共享变量的副本。这会产生一个问题：一个处理器对其高速缓存中副本数据的更新，其他处理器无法知晓，导致其他处理器无法正确读取该共享变量，这就是缓存一致性问题，实质就是如何防止读脏数据和丢失更新的问题。为了解决该问题，处理器之间需要一种通信机制——**缓存一致性协议**（Cache Coherence Protocal）。缓存一致性协议常见的有 MESI（Modified-Exclusive-Shared-Invalid） 协议。

## 写缓冲区与无效化队列

MESI 协议解决了缓存一致性问题，但其自身也存在性能问题——处理器在执行写内存操作，会进入阻塞状态，必须等待其他所有处理器将其自身高速缓存中的副本数据删除并接受到删除的消息。导致了处理器的等待和延迟，因此引入了**写缓冲区（Store Buffer 或 Write Buffer）**和**无化队列（Invalidate Queue）**来避免和减少该问题的发生。写缓冲区和无效化队列的引入又会带来新问题——内存重排序和可见性问题。

写缓冲区和无效化队列都可能导致**内存重排序**。写缓冲区可能导致 StoreLoad 重排序（Stores Reordered After Loads）和 StoreStore 重排序（Stores Reordered After Stores），无效化队列可能会导致 LoadLoad 重排序（Loads Reordered After Loads）。不同处理器架构所支持的内存重排序不同。比如，现代处理器都会采用写缓冲区，而有的处理器（x86）会保障写操作的顺序，即不允许 StoreStore 重排序。

写缓冲区是处理器内部的私有高速存储部件，处理器之间无法相互读取写缓冲区中存储的内存，因此，可能会导致可见性问题，并且作为**可见性问题**的根本根源。为了保证一个处理器对共享变量所做的修改可以被其他处理器同步，编译器等底层系统需要借助一类被称为**内存屏障**的特殊指令。内存屏障中的**存储屏障**可以使执行该指令的处理器冲刷其写缓冲区。可见性问题的另一半是无效化队列导致的，内存屏障中的**加载屏障**可以用来解决无效化队列造成的可见性问题。

## 基本内存屏障

处理器支持哪种内存重排序（LoadLoad 重排序、LoadStore 重排序、StoreStore 重排序和 StoreLoad 重排序）就会提供能够禁止该重排序的指令，这些指令就被称为**基本内存屏障**（LoadLoad 屏障、LoadStore 屏障、StoreStore 屏障和 StoreLoad 屏障）。

基本内存屏障是对这一类指令的称呼，这类指令确保该指令左侧的任何 Store/Load 操作与指令右侧的 Store/Load 之间进行重排序。从而确保该指令左侧的所有 Store/Load 操作先于指令右侧的 Store/Load 操作被提交，即内存操作作用到主内存（或高速缓存上）。

基本内存屏障的具体作用如下表。

| 屏障类型   | 指令示例                  | 具体作用                                                     |
| ---------- | ------------------------- | ------------------------------------------------------------ |
| StoreLoad  | Store1;StoreLoad;Load1;   | 禁止 StoreLoad 重排序。确保该屏障前的任意写操作（Store1）都会在该屏障后的任意读操作（Load1）的数据被加载之前同步到处理器。 |
| StoreStore | Store1;StoreStore;Store2; | 禁止 StoreStore 重排序。确保该屏障前的任意写操作（Store1）都会在该屏障后的任意写操作（Store2）之前同步到处理器。 |
| LoadLoad   | Load1;LoadLoad;Load2;     | 禁止 LoadLoad 重排序。确保该屏障前的任意读操作（Load1）的数据都会在该屏障后的任意读操作（Load1）之前被加载到处理器。 |
| LoadStore  | Load1;LoadStore;Store1;   | 禁止 LoadStore 重排序。确保该屏障前的任意读操作（Load1）的数据都会在该屏障后的任意写操作（Store1）的数据冲刷到高速缓存（或主内存）之前被加载到处理器。 |

屏障两侧的内存操作仍可以在不越过内存屏障本身的情况下进行重排序。并且 XY 屏障左侧的非 X 操作也可以与屏障右侧的非 Y 操作进行重排序。如下所示，因为 StoreLoad 屏障的存在，屏障左侧的 Store1、Store2 无法与右侧的 Load2、Load3 进行重排序。但左侧的 Store1、Load1、Store2 与右侧的 Store3、Load2、Load3 都可以各自进行重排序，并且左侧的 Load1 与右侧的 Store1 也可以进行重排序。

```
Store1;Load1;Store2; StoreLoad 屏障; Store3;Load2;Load3
```

编译器（JIT 编译器）、运行时（JVM）和处理器都会遵守内存屏障，保障其作用得以落实。LoadLoad 屏障是通过清空无效化队列来实现禁止 LoadLoad  重排序，StoreStore 屏障是通过对写缓冲区中的条目进行标记来实现禁止 StoreStore 重排序。

StoreLoad 屏障是一个万能屏障，兼具其他 3 种基本内存屏障的效果，现代处理器大多支持该屏障，但是开销也是最大的， StoreLoad 会清空无效化队列，将写缓冲区中的数据全部写入主内存中。Java 虚拟机对 synchronized、volatile 和 final 同步原语的实现就是基于内存屏障的。

## 处理器优化

除了在处理器和主内存之间增加高速缓存外，为了提高使处理器内部的运算单元能尽量被使用，处理器可能会对输入的代码进行乱序执行，虽然保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致，因此如果多个计算任务相互依赖，则执行顺序不能依靠代码的先后顺序顺序保证，这就是**处理器优化**。与处理器的乱序执行优化类似，Java 虚拟机的即时编译器也有指令重排序优化。

# 并发问题的源头



# Java 内存模型

## 什么是内存模型

在处理器内存模型中，我们了解了处理器内存模型的由来，那么什么是内存模型？从底层的角度看，计算机需要解决：一个处理器对其共享变量的更新在什么情况下会被其他处理器知晓，即**可见性**问题。可见性问题衍生出**有序性**问题，即一个处理器在先后更新这些共享变量的情况下，其他处理器是怎么按顺序读取这些更新的变量的。用于回答上述问题的模型即为**内存一致性模型（Memory Consistency Model）**，也被称为**内存模型（Momory Model）**。不同的处理器有着不同的内存模型，内存模型就是为了解决并发编程中共享内存的正确性而定义的规则。主要通过**限制处理器优化**和**使用内存屏障**来解决并发问题。

## 什么是 Java 内存模型

Java 作为一门跨平台的语言，为了使 Java 开发者不用关心不同处理器上内存模型的差异，Java 定义了自己的内存模型，该模型称为 Java 内存模型（Java Memory Model，简称 JMM）。

JMM 是一个语言级的内存模型。JMM 是一组规范，需要各个 JVM 的实现来遵守 JMM 规范，如果没有这样的一个 JMM 规范，那么 Java 程序可能在经过不同 JVM 的不同规则重排序后，导致在不同 JVM 上的运行结果不一样，JMM 屏蔽了不同硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。我们说的 Java 内存模型一般指 JSR-133（JDK5） 修正后的内存模型。

JMM 中定义了 final、synchronized 和 volaile 同步原语来保证正确同步的 Java 程序能够正确的运行在不同的硬件或操作系统上，如果没有 JMM，就需要我们自己指定什么时候用哪些内存屏障，太麻烦了，有了 JMM 后，我们只要用这些同步原语就够了

## Java 内存模型的实现

### JMM 的抽象模型

在 Java 中，所有实例域、静态域和数组元素（共享变量）都存储在堆元素中，堆内存在线程之间共享。局部变量、方法定义参数和异常处理器参数是线程私有的，不存在内存可见性问题，不受内存模型影响。

Java 线程之间的通信通过 JMM 控制，JMM 是对硬件内存模型的抽象，从抽象的角度看，JMM 定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存，每个线程都有自己的本地内存，本地内存中存储了共享变量读/写的副本。本地内存是 JMM 的一个抽象概念，不是真实存在的，它包含了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。Java 内存模型的抽象如下如所示。

![image-20200210155034567](C:\Users\hncboy\AppData\Roaming\Typora\typora-user-images\image-20200210155034567.png)

### JMM 与硬件内存模型的关系

从 JDK1.3 起，“主流”平台上的“主流”商用 Java 虚拟机的线程模型普遍都被替换为基于操作系统原生线程模型来实现的，即采用 1:1 的线程模型。因此我们可以知道，多线程的执行最终都会被映射到硬件处理器上执行。对硬件来说，只有寄存器、高速缓存、主内存等概念，并没有本地内存和主内存之分。JMM 只是一个抽象的内存模型，并不是真正存在这样的空间，只是通过这种方式去表达出来更容易理解，本质上是定义线程和主内存之间的交互方法，JMM 和硬件层面的内存模型对应关系如下图所示。

### 可见性和重排序问题

从硬件内存模型中我们知道了写缓冲区会存在内存可见性问题，处理器对指令进行优化时，会有重排序问题。JMM 私有的本地内存也会有可见性问题，而重排序除了存在处理器重排序，还存在编译器重排序。一般重排序分为 3 种类型。

- 编译器优化的重排序：编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
- 指令级并行的重排序：现代处理器采用了指令级并行技术（Instruction Level Parallelism）来将多条指令重叠执行。如果不存在数据依赖，处理器可以改变语句对应机器的执行顺序。
- 内存系统的重排序：由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

![image-20200201164449476](E:/Project/GithubProject/StudyNotes/book/Java并发编程的艺术/pics/image-20200201164449476.png)

其中 1 属于编译器重排序，2 和 3 属于处理器重排序。对于编译器重排序，JMM 的编译器重排序规则会**禁止特定类型**的编译器重排序）。对于处理器重排序，JMM 的处理器重排序规则会要求 Java 编译器在生成指令时，插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的处理器重排序。

本地内存的可见性问题即 CPU 缓存造成的可见性问题，这属于硬件层面的问题，语言层面无法解决，而我们可以用过禁止特定的重排序来解决内存可见性问题。

### 可见性和重排序解决方案

JMM 没有限制执行引擎使用处理器的特定寄存器或缓存来和主内存进行交互，也没有限制编译器是否要进行调整代码执行顺序这类优化措施，也就是说，在 JMM 中，它会存在内存可见性和重排序问题，只是 JMM 把底层的问题抽象到了 JVM 层面，再基于 CPU 层面提供的内存屏障指令以及限制编译器重排序的一些操作来解决并发问题。