

# Java虚拟机技术笔记



## Java虚拟机内存分布

![image-20190927162252866](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162252866.png)

## 各部分内存

- (线程隔离) **程序计数器**（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能 会通过一些更高效的方式去实现），**字节码解释器工作时就是通过改变这个计数器的值来选 取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需 要依赖这个计数器来完成。**由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的， 在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私 有”的内存。如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指 令的地址；如果正在执行的是Native方法，这个计数器值则为空（Undefined）。**此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。**
- (线程隔离) **Java虚拟机栈**（Java Virtual Machine Stacks）也是线程私有的，**它的生命周期与线程相同。**虚拟机栈描述的是Java方法执行的内存模型：**每个方法在执行的同时 都会创建一个栈帧（Stack Frame ）用于存储局部变量表、操作数栈、动态链接、方法出口 等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出 栈的过程。**局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、 float、long、double）、对象引用（reference类型，它不等同于对象本身，可能是一个指向对 象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）和 returnAddress类型（指向了一条字节码指令的地址）。其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据 类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这 个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变 量表的大小。在Java虚拟机规范中，对这个区域规定了两种异常状况：**如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），如 果扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常。**
- (线程隔离) **本地方法栈**（Native Method Stack）与虚拟机栈所发挥的作用是非常相似的，它们之间 的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而**本地方法栈则为虚拟机使用到的Native方法服务**。在虚拟机规范中对本地方法栈中方法使用的语言、使用方式 与数据结构并没有强制规定，因此具体的虚拟机可以自由实现它。甚至有的虚拟机（譬如 Sun HotSpot虚拟机）直接就把本地方法栈和虚拟机栈合二为一。**与虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。**
- (线程共享) **Java堆**是被所有线程共享的一块内存区域，在虚拟机启动时创建。**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。**这一点在Java虚拟机规范中的描 述是：所有的对象实例以及数组都要在堆上分配 **Java堆是垃圾收集器管理的主要区域**，因此很多时候也被称做“GC堆”. 由于现在收集器基 本都采用分代收集算法，所以**Java堆中还可以细分为：新生代和老年代；再细致一点的有 Eden空间、From Survivor空间、To Survivor空间等。**从内存分配的角度来看，线程共享的 Java堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer,TLAB）。不过无论如何划分，都与存放内容无关，无论哪个区域，存储的都仍然是对象实例，进一步划分的目的是为了更好地回收内存，或者更快地分配内存。根据Java虚拟机规范的规定，**Java堆可以处于物理上不连续的内存空间中，只要逻辑上 是连续的即可**，就像我们的磁盘空间一样。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前**主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。如 果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异 常。**

- (线程共享) **方法区**（Method Area）与Java堆一样，是各个线程共享的内存区域，**它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。**虽然Java虚拟机规 范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应 该是与Java堆区分开来。Java虚拟机规范对方法区的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。相对而言，**垃圾收集行为在这个区域是比较少出现的，但并非数据进入了方法区就如永久代的名字一样“永久”存在了。这区域的内存回收目标主要是针对常量池的回收和对类型的卸载**，一般来说，这个区域的回 收“成绩”比较难以令人满意，尤其是类型的卸载，条件相当苛刻，但是这部分区域的回收确 实是必要的。**根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出 OutOfMemoryError异常。Java8中已经废弃了永久代,改为元空间,它和永久代的区别就是它的数据不存放在java虚拟机的内存中,而是直接存放在宿主机内存中**

- (线程共享) **运行时常量池**（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table），**用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。**Java虚拟机对Class文件每一部分（自然也包括常量池）的格式都有严格规定，每一个字节用于存储哪种数据都必须符合规范上的要求才会被虚拟机认可、装载和执行，但对于运行时常量池，Java虚拟机规范没有做任何细节的要求，不同的提供商实现的虚拟机可以按照自己的需要来实现这个内存区域。不过，一般来说，除了保存Class文件中描述的符号引用外， 还会把翻译出来的直接引用也存储在运行时常量池中 。运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern（）方法。**既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。**

- 直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError 异常出现  本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，肯定还是会受到本机总内存（包括RAM以及SWAP区或者分页文件）大小以及处理器寻址空间的限 制。服务器管理员在配置虚拟机参数时，会根据实际内存设置-Xmx等参数信息，但经常忽略直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制）， 从而导致动态扩展时出现OutOfMemoryError异常。



##  对象创建过程

- 检测类是否被加载: 当虚拟机执行到new时，会先去常量池中查找这个类的符号引用。如果能找到符号引用，说明此类已经被加载到方法区(方法区存储虚拟机已经加载的类的信息),可以继续执行；如果找不到符号引用，就会使用类加载器执行类的加载过程，类加载完成后继续执行。

- 为对象分配内存: 在虚拟机中的堆上划分出一块内存，存储类的对象（大小在类加载完成后，根据其内部的变量类型与引用就可以确定类需要的空间大小）。由于GC的机制不同，所以**在分配时可能存在两种机制**。**如果垃圾回收机制为标记清除法那么虚拟机必须维护一个列表，记录着哪一块内存可用，再分配时找到一块足够大的空间，划分给类对象实例。如果垃圾回收机制为复制算法或标记整理法，那么在分配内存时，直接将指针向空闲区域移动一段与对象大小相等的距离即可。**在内存分配时，解决多线程同步的问题，有两种方案：一种是对分配内从空间的动作做同步处理；另一种是把内存分配按照线程划分，即每个线程在Java堆中预先分配一小块内存，成为本地线程分配缓冲。

- 对象的内存分配完成后，还需要**将对象的内存空间都初始化为零值**，这样能保证对象即使没有赋初值，也可以直接使用。
- 分配完内存空间，初始化零值之后，**虚拟机还需要对对象进行其他必要的设置，设置的地方都在对象头中，包括这个对象所属的类，类的元数据信息，对象的hashcode，GC分代年龄等信息**。

- **执行init方法, 进行对象初始化**



##  常见的内存溢出错误

- Java堆溢出: 是实际应用中常见的内存溢出情况。**当出现Java堆内存溢出 时，异常堆栈信息“java.lang.OutOfMemoryError”会跟着进一步提示“Java heap space”。**通过参数-XX： +HeapDumpOnOutOfMemoryError可以让虚拟机在出现内存溢出异常时Dump出当前的内存堆 转储快照以便事后进行分析。要解决这个区域的异常，一般的手段是先通过内存映像分析工具（如Eclipse Memory Analyzer）对Dump出来的堆转储快照进行分析，重点是确认内存中的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）**内存泄漏指的是对象已经没用,但是由于存在引用链, 没有被垃圾回收器进行回收, 堆积起来造成内存泄漏。而内存溢出则是由于分配给虚拟机的内存不足**, 此时应当检查虚拟机的堆参数（-Xmx与-Xms），与机器物理内存对比看是否还可以调大，从代码上检查是 否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。

- 虚拟机栈和本地方法栈溢出: HotSpot的虚拟机中栈的容量由-Xss参数设定(每个线程的栈大小)。在Java虚拟机规范中描述了两种异常：

  - 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常。

  - 如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常。

  这里把异常分成两种情况，看似更加严谨，但却存在着一些互相重叠的地方：当栈空间无法继续分配时，到底是内存太小，还是已使用的栈空间太大，其本质上只是对同一件事情 的两种描述而已。

  实验结果在单个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。如果测试时不限于单线程，通过不断地建立线程的方式倒是可以产生内存溢出异常。出现StackOverflowError异常时有错误堆栈可以阅读，相对来说，比较容易找到问题的所在。而且，如果使用虚拟机默认参数，栈 深度在大多数情况下（因为每个方法压入栈的帧大小并不是一样的，所以只能说在大多数情 况下）达到1000～2000完全没有问题，对于正常的方法调用（包括递归），这个深度应该完 全够用了。但是，**如果是建立过多线程导致的内存溢出，在不能减少线程数或者更换64位虚拟机的情况下，就只能通过减少最大堆和减少单个线程最大的栈容量来换取更多的线程。**如果没有这方面的处理经验，这种通过“减少内存”的手段来解决内存溢出的方式会比较难以想到。

- 方法区和运行时常量池溢出: 可以用String.intern()方法进行方法区的内存溢出测试, **这个方法的作用是如果字符串常量池中已经包含一个等 于此String对象的字符串，则返回代表池中这个字符串的String对象；否则，将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。**运行时常量池内存溢出在OutOfMemoryError后面跟随的提示信息 是“PermGen space”，说明运行时常量池属于方法区（HotSpot虚拟机中的永久代）的一部 分。**(JDK1.7之后逐步移除永久代,不会抛出永久代内存溢出异常)**

- 本机直接内存溢出: DirectMemory容量可通过-XX：MaxDirectMemorySize指定，如果不指定，则默认与Java 堆最大值（-Xmx指定）一样，由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，**如果发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就 可以考虑检查一下是不是这方面的原因。**



## 判断对象是否存活的算法

- **引用计数算法**: 给对象中添加一个引用计数器，每当有 一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0 的对象就是不可能再被使用的。但是，至少主流 的Java虚拟机里面没有选用引用计数算法来管理内存，其中最主要的原因是**它很难解决对象 之间相互循环引用的问题**。例子: 对象objA和objB都有字段 instance，赋值令objA.instance=objB及objB.instance=objA，除此之外，这两个对象再无任何引 用，实际上这两个对象已经不可能再被访问，但是它们因为互相引用着对方，导致它们的引 用计数都不为0，于是引用计数算法无法通知GC收集器回收它们。

- **可达性分析算法**: 这个算法的基本思 路就是通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），**当一个对象到GC Roots没有任何引用链相连 （用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。**如 图所示，对象object 5、object 6、object 7虽然互相有关联，但是它们到GC Roots是不可达的，所以它们将会被判定为是可回收的对象。



![image-20190927162311856](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162311856.png)

  在Java语言中，可作为GC Roots的对象包括下面几种：

  - 虚拟机栈（栈帧中的本地变量表）中引用的对象。

  - 方法区中类静态属性引用的对象。

  - 方法区中常量引用的对象。

  - 本地方法栈中JNI（即一般说的Native方法）引用的对象。



## JVM中四个级别的引用

在JDK 1.2之后，Java对引用的概念进行了扩充，将引用分为强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）、虚引用（Phantom Reference）4种，这4种引用强度依次逐渐减弱。

- **强引用**就是指在程序代码之中普遍存在的，类似“Object obj=new Object（）”这类的引 用，只要强引用还存在，垃圾收集器**永远不会回收掉被引用的对象。**

- **软引用**是用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，**在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。**如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK 1.2之后，提供了SoftReference类来实现软引用。

- **弱引用**也是用来描述非必需对象的，但是它的强度比软引用更弱一些，**被弱引用关联的对象只能生存到下一次垃圾收集发生之前**。当垃圾收集器工作时，无论当前内存是否足够， 都会回收掉只被弱引用关联的对象。在JDK 1.2之后，提供了WeakReference类来实现弱引用。

- **虚引用**也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也**无法通过虚引用来取得一个对象实例**。为一 个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。在 JDK 1.2之后，提供了PhantomReference类来实现虚引用。

详细解析: <https://blog.csdn.net/l540675759/article/details/73733763>



## java中的finalize方法

即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选， **筛选的条件是此对象是否有必要执行finalize（）方法。当对象没有覆盖finalize（）方法，或 者finalize（）方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。**

**如果这个对象被判定为有必要执行finalize（）方法，那么这个对象将会放置在一个叫做 F-Queue的队列之中，并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行 它。这里所谓的“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束，**这样做的原因是，如果一个对象在finalize（）方法中执行缓慢，或者发生了死循环（更极端的情 况），将很可能会导致F-Queue队列中其他对象永久处于等待，甚至导致整个内存回收系统崩溃。finalize（）方法是对象逃脱死亡命运的最后一次机会，稍后GC将对F-Queue中的对象 进行第二次小规模的标记，如果对象要在finalize（）中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的 成员变量，那在第二次标记时它将被移除出“即将回收”的集合；如果对象这时候还没有逃 脱，那基本上它就真的被回收了。

**Java不推荐使用finalize方法,因为一个对象什么时候被回收是不确定的, 因此finalize方法何时被调用, 是否一定会被调用都是不确定的。**

不象 C++ 中的析构函数，Java Applet 不会自动执行你的类中的finalize() 方法。事实上，如果你正在使用 Java 1.0，即使你试图强制它调用finalize() 方法，也不能确保将调用它。

**因此，你不应当依靠finalize() 来执行你的 Applet 和应用程序的资源清除工作。取而代之，你应当明确的清除那些资源或创建一个try...finally 块（或类似的机制）来实现。**



## 方法区回收

**永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类。**回收废弃常量与回收 Java堆中的对象非常类似。以常量池中字面量的回收为例，假如一个字符串“abc”已经进入了常量池中，但是当前系统没有任何一个String对象是叫做“abc”的，换句话说，就是没有任何 String对象引用常量池中的“abc”常量，也没有其他地方引用了这个字面量，如果这时发生内存回收，而且必要的话，这个“abc”常量就会被系统清理出常量池。常量池中的其他类（接 口）、方法、字段的符号引用也与此类似。

判定一个常量是否是“废弃常量”比较简单，而要判定一个类是否是“无用的类”的条件则 相对苛刻许多。**类需要同时满足下面3个条件才能算是“无用的类”：**

- - **该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例。**

- - **加载该类的ClassLoader已经被回收。**
  - **该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该 类的方法。**

虚拟机可以对满足上述3个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样，不使用了就必然会回收。**是否对类进行回收，HotSpot虚拟机提供了-Xnoclassgc 参数进行控制**，还可以使用-verbose：class以及-XX：+TraceClassLoading、-XX： +TraceClassUnLoading查看类加载和卸载信息，其中-verbose：class和-XX： +TraceClassLoading可以在Product版的虚拟机中使用，-XX：+TraceClassUnLoading参数需要 FastDebug版的虚拟机支持。**在大量使用反射、动态代理、CGLib等ByteCode框架、动态生成JSP以及OSGi这类频繁自定义ClassLoader的场景都需要虚拟机具备类卸载的功能，以保证永久代不会溢出。**	



## 垃圾收集算法

### 1. 标记-清除算法

最基础的收集算法是“标记-清除”（Mark-Sweep）算法，如同它的名字一样，算法分 为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有 被标记的对象，它的标记过程其实在前一节讲述对象标记判定时已经介绍过了。之所以说它是最基础的收集算法，是因为后续的收集算法都是基于这种思路并对其不足进行改进而得到 的。**它的主要不足有两个：一个是效率问题，标记和清除两个过程的效率都不高；另一个是 空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程 序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。**

![image-20190927162323333](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162323333.png)

### 2. 复制算法

为了解决效率问题，一种称为“复制”（Copying）的收集算法出现了，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着 的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是 对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。只是这种算法的代价是将内存缩小为了原来的一半，未免太高了一点。

![image-20190927162331585](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162331585.png)

现在的商业虚拟机都采用这种收集算法来回收新生代，IBM公司的专门研究表明，新生 代中的对象98%是“朝生夕死”的，所以并不需要按照1:1的比例来划分内存空间，而是**将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor 。 当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。**HotSpot虚拟机默认Eden和Survivor的大小比例是 8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%（80%+10%），只有10% 的内存会被“浪费”。当然，98%的对象可回收只是一般场景下的数据，我们没有办法保证每次回收都只有不多于10%的对象存活，**当Survivor空间不够用时，需要依赖其他内存（这里 指老年代）进行分配担保（HandlePromotion）**。

![image-20190927162339333](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162339333.png)

### 3. 标记-整理算法

**复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。**更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。

根据老年代的特点，有人提出了另外一种“标记-整理”（Mark-Compact）算法，标记过程 仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存 活的对象都向一端移动，然后直接清理掉端边界以外的内存，“标记-整理”算法的示意图如图所示。

![image-20190927162347755](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162347755.png)

### 4. 分代收集算法

当前商业虚拟机的垃圾收集都采用“分代收集”（Generational Collection）算法，这种算 法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。**一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代 中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间 对它进行分配担保，就必须使用“标记—清理”或者“标记—整理”算法来进行回收。**



## 可达性分析过程

从可达性分析中从GC Roots节点找引用链这个操作为例，可作为GC Roots的节点主要在 全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）中，现 在很多应用仅仅方法区就有数百兆，如果要逐个检查这里面的引用，那么必然会消耗很多时间。

可达性分析对执行时间的敏感还体现在GC停顿上，因为这项分析工作必须在一个能确保一致性的快照中进行——这里“一致性”的意思是指在整个分析期间整个执行系统看起来就像被冻结在某个时间点上，不可以出现分析过程中对象引用关系还在不断变化的情 况，该点不满足的话分析结果准确性就无法得到保证。**这点是导致GC进行时必须停顿所有 Java执行线程（Sun将这件事情称为“Stop The World”）的其中一个重要原因，即使是在号称 （几乎）不会发生停顿的CMS收集器中，枚举根节点时也是必须要停顿的。**

由于目前的主流Java虚拟机使用的都是准确式GC，所以当执行系统停顿下来后，并不需要一个不漏地检查完所有 执行上下文和全局的引用位置，虚拟机应当是有办法直接得知哪些地方存放着对象引用。**在 HotSpot的实现中，是使用一组称为OopMap的数据结构来达到这个目的的，在类加载完成的时候，HotSpot就把对象内什么偏移量上是什么类型的数据计算出来，在JIT编译过程中，也 会在特定的位置记录下栈和寄存器中哪些位置是引用。**



## 安全点

在OopMap的协助下，HotSpot可以快速且准确地完成GC Roots枚举，但一个很现实的问题随之而来：可能导致引用关系变化，或者说OopMap内容变化的指令非常多，如果为每一 条指令都生成对应的OopMap，那将会需要大量的额外空间，这样GC的空间成本将会变得很高。

实际上，HotSpot也的确**没有为每条指令都生成OopMap**，前面已经提到，**只是在“特定的位置”记录了这些信息，这些位置称为安全点（Safepoint）**，**即程序执行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停。**Safepoint的选定既不能太少以致于让 GC等待时间太长，也不能过于频繁以致于过分增大运行时的负荷。所以，安全点的选定基 本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的——因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这个原因而过长时间运行，**“长时间执行”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有 这些功能的指令才会产生Safepoint。**

对于Sefepoint，另一个需要考虑的问题是**如何在GC发生时让所有线程（这里不包括执行 JNI调用的线程）都“跑”到最近的安全点上再停顿下来。这里有两种方案可供选择：抢先式中断（Preemptive Suspension）和主动式中断（Voluntary Suspension），其中抢先式中断不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程 中断的地方不在安全点上，就恢复线程，让它“跑”到安全点上。现在几乎没有虚拟机实现采用抢先式中断来暂停线程从而响应GC事件**。

**而主动式中断的思想是当GC需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起。 轮询标志的地方和安全点是重合的，另外再加上创建对象需要分配内存的地方。**



## 安全区域

使用Safepoint似乎已经完美地解决了如何进入GC的问题，但实际情况却并不一定。 Safepoint机制保证了程序执行时，在不太长的时间内就会遇到可进入GC的Safepoint。但是， 程序“不执行”的时候呢？所谓的程序不执行就是没有分配CPU时间，典型的例子就是线程处 于Sleep状态或者Blocked状态，这时候线程无法响应JVM的中断请求，“走”到安全的地方去 中断挂起，JVM也显然不太可能等待线程重新被分配CPU时间。对于这种情况，就需要安全 区域（Safe Region）来解决。

安全区域是指在一段代码片段之中，引用关系不会发生变化。在这个区域中的任意地方 开始GC都是安全的。我们也可以把Safe Region看做是被扩展了的Safepoint。

在线程执行到Safe Region中的代码时，首先标识自己已经进入了Safe Region，那样，**当在这段时间里JVM要发起GC时，就不用管标识自己为Safe Region状态的线程了。在线程要离开Safe Region时，它要检查系统是否已经完成了根节点枚举（或者是整个GC过程），如果完成了，那线程就继续执行，否则它就必须等待直到收到可以安全离开Safe Region的信号为止。**



## 垃圾收集器

![image-20190927162358691](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162358691.png)

上图展示了7种作用于不同分代的收集器，如果两个收集器之间存在连线，就说明它们可以搭配使用。虚拟机所处的区域，则表示它是属于新生代收集器还是老年代收集器。

虽然我们是在对各个收集器进行比较，但并非为了挑选出一个最好的收集器。因为直到现在为止还没有最好的收集器 出现，更加没有万能的收集器，所以我们选择的只是对具体应用最合适的收集器。这点不需要多加解释就能证明：如果有一种放之四海皆准、任何场景下都适用的完美收集器存在，那 HotSpot虚拟机就没必要实现那么多不同的收集器了。



### 1. Serial收集器

**Serial收集器**是最基本、发展历史最悠久的收集器，曾经（在JDK 1.3.1之前）是虚拟机新生代收集的唯一选择。大家看名字就会知道，这个收集器是一个单线程的收集器，但它 的“单线程”的意义并不仅仅说明它只会使用一个CPU或一条收集线程去完成垃圾收集工作， 更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。“Stop The World”这个名字也许听起来很酷，但这项工作实际上是由虚拟机在后台自动发起和自动 完成的，在用户不可见的情况下把用户正常工作的线程全部停掉，这对很多应用来说都是难以接受的。

![image-20190927162425112](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162425112.png)

实际上到现在为止，**它依然是虚拟机运行在Client模式下的默认新生代收集器**。 它也有着优于其他收集器的地方：简单而高效（与其他收集器的单线程比），对于限定单个 CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最 高的单线程收集效率。在用户的桌面应用场景中，分配给虚拟机管理的内存一般来说不会很大，收集几十兆甚至一两百兆的新生代（仅仅是新生代使用的内存，桌面应用基本上不会再大了），停顿时间完全可以控制在几十毫秒最多一百多毫秒以内，只要不是频繁发生，这点停顿是可以接受的。所以，Serial收集器对于运行在Client模式下的虚拟机来说是一个很好的选择。



### 2. ParNew收集器

**ParNew收集器其实就是Serial收集器的多线程版本**，除了使用多条线程进行垃圾收集之 外，其余行为包括Serial收集器可用的所有控制参数（例如：-XX：SurvivorRatio、-XX： PretenureSizeThreshold、-XX：HandlePromotionFailure等）、收集算法、Stop The World、对象分配规则、回收策略等都与Serial收集器完全一样，在实现上，这两种收集器也共用了相 当多的代码。ParNew收集器除了多线程收集之外，其他与Serial收集器相比并没有太多创新之处，但**它却是许多运行在Server模式下的虚拟机中首选的新生代收集器**，其中有一个与性能无关但很重要的原因是，**除了Serial收集器外，目前只有它能与CMS收集器配合工作。**

![image-20190927162434937](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162434937.png)

ParNew收集器在单CPU的环境中绝对不会有比Serial收集器更好的效果，甚至由于存在线程交互的开销，该收集器在通过超线程技术实现的两个CPU的环境中都不能百分之百地保证可以超越Serial收集器。当然，随着可以使用的CPU的数量的增加，它对于GC时系统资源的有效利用还是很有好处的。它默认开启的收集线程数与CPU的数量相同，在CPU非常多 （譬如32个，现在CPU动辄就4核加超线程，服务器超过32个逻辑CPU的情况越来越多了）的 环境下，可以使用-XX：ParallelGCThreads参数来限制垃圾收集的线程数。



### 3. Parallel Scavenge收集器

Parallel Scavenge收集器是一个新生代收集器，它也是使用复制算法的收集器，又是并行的多线程收集器……看上去和ParNew都一样，那它有什么特别之处呢？

Parallel Scavenge收集器的特点是它的关注点与其他收集器不同，**CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge收集器的目标则是达到 一个可控制的吞吐量（Throughput）**。所谓**吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间）**，虚 拟机总共运行了100分钟，其中垃圾收集花掉1分钟，那吞吐量就是99%。

**停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验，而高吞吐量则可以高效率地利用CPU时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。**

Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集 停顿时间的-XX：MaxGCPauseMillis参数以及直接设置吞吐量大小的-XX：GCTimeRatio参 数。

MaxGCPauseMillis参数允许的值是一个大于0的毫秒数，收集器将尽可能地保证内存回 收花费的时间不超过设定值。不过大家不要认为如果把这个参数的值设置得稍小一点就能使 得系统的垃圾收集速度变得更快，GC停顿时间缩短是以牺牲吞吐量和新生代空间来换取 的：系统把新生代调小一些，收集300MB新生代肯定比收集500MB快吧，这也直接导致垃圾 收集发生得更频繁一些，原来10秒收集一次、每次停顿100毫秒，现在变成5秒收集一次、每 次停顿70毫秒。停顿时间的确在下降，但吞吐量也降下来了。

GCTimeRatio参数的值应当是一个大于0且小于100的整数，也就是垃圾收集时间占总时 间的比率，相当于是吞吐量的倒数。如果把此参数设置为19，那允许的最大GC时间就占总 时间的5%（即1/（1+19）），默认值为99，就是允许最大1%（即1/（1+99））的垃圾收集 时间。

由于与吞吐量关系密切，Parallel Scavenge收集器也经常称为“吞吐量优先”收集器。除上 述两个参数之外，Parallel Scavenge收集器还有一个参数-XX：+UseAdaptiveSizePolicy值得关 注。这是一个开关参数，当这个参数打开之后，就不需要手工指定新生代的大小（-Xmn）、 Eden与Survivor区的比例（-XX：SurvivorRatio）、晋升老年代对象年龄（-XX： PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信 息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种调节方式称为GC 自适应的调节策略（GC Ergonomics） 。如果读者对于收集器运作原来不太了解，手工优化 存在困难的时候，使用Parallel Scavenge收集器配合自适应调节策略，把内存管理的调优任务 交给虚拟机去完成将是一个不错的选择。只需要把基本的内存数据设置好（如-Xmx设置最大 堆），然后使用MaxGCPauseMillis参数（更关注最大停顿时间）或GCTimeRatio（更关注吞 吐量）参数给虚拟机设立一个优化目标，那具体细节参数的调节工作就由虚拟机完成了。自适应调节策略也是Parallel Scavenge收集器与ParNew收集器的一个重要区别。



### 4. Serial Old收集器

**Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用“标记-整 理”算法**。**这个收集器的主要意义也是在于给Client模式下的虚拟机使用。**如果在Server模式 下，那么它主要还有两大用途：一种用途是在JDK 1.5以及之前的版本中与Parallel Scavenge 收集器搭配使用 ，另一种用途就是作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用。

![image-20190927162447862](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162447862.png)



### 5. Parallel Old收集器

**Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。** 这个收集器是在JDK 1.6中才开始提供的，在此之前，新生代的Parallel Scavenge收集器一直 处于比较尴尬的状态。原因是，如果新生代选择了Parallel Scavenge收集器，老年代除了 Serial Old（PS MarkSweep）收集器外别无选择。由于老年代Serial Old收集器在服务端应用性能上的“拖累”，使用了Parallel Scavenge收集器也未必能在整体应用上获得吞吐量最大化的效果，由于单线程的老年代收集中无法充分利用服务器多CPU的处理能力，在老年代很大而且硬件比较 高级的环境中，这种组合的吞吐量甚至还不一定有ParNew加CMS的组合“给力”。**直到Parallel Old收集器出现后，“吞吐量优先”收集器终于有了比较名副其实的应用组 合，在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge加Parallel Old 收集器。**

![image-20190927162454695](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162454695.png)

### 6. CMS收集器(Concurrent Mark Sweep)

**CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。** **目前很大一部分的Java应用集中在互联网站或者B/S系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最短，以给用户带来较好的体验。CMS收集器就非常符合这类应用的需求。**

从名字（包含“Mark Sweep”）上就可以看出，**CMS收集器是基于“标记—清除”算法实现的**，它的运作过程相对于前面几种收集器来说更复杂一些，整个过程分为4个步骤，包括：

- 初始标记（CMS initial mark） 

- 并发标记（CMS concurrent mark） 

- 重新标记（CMS remark） 

- 并发清除（CMS concurrent sweep） 

其中，**初始标记、重新标记这两个步骤仍然需要“Stop The World”。**初始标记仅仅只是 标记一下GC Roots能直接关联到的对象，速度很快，并发标记阶段就是进行GC RootsTracing 的过程，而重新标记阶段则是为了修正并发标记期间因用户程序继续运作而导致标记产生变 动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。

由于整个过程中耗时最长的并发标记和并发清除过程收集器线程都可以与用户线程一起工作，所以，**从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。**

![image-20190927162502196](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162502196.png)

CMS的缺点:

- **CMS收集器对CPU资源非常敏感**。**在并发阶段，它虽然不会导致用户线程停顿，但是会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。**CMS默认启动的回收线程数是（CPU数量 +3）/4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且 随着CPU数量的增加而下降。但是当CPU不足4个（譬如2个）时，CMS对用户程序的影响就 可能变得很大，如果本来CPU负载就比较大，还分出一半的运算能力去执行收集器线程，就 可能导致用户程序的执行速度忽然降低了50%
- **CMS收集器无法处理浮动垃圾（Floating Garbage）**，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这一部分垃圾出现在标记过程之后，CMS无法 在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就称为“浮动垃圾”。也是由于在垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此**CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进 行收集，需要预留一部分空间提供并发收集时的程序运作使用**。在JDK 1.5的默认设置 下，CMS收集器当老年代使用了68%的空间后就会被激活，这是一个偏保守的设置，如果在应用中老年代增长不是太快，可以适当调高参数-XX：CMSInitiatingOccupancyFraction的值来 提高触发百分比，以便降低内存回收次数从而获取更好的性能，在JDK 1.6中，CMS收集器 的启动阈值已经提升至92%。**要是CMS运行期间预留的内存无法满足程序需要，就会出现一 次“Concurrent Mode Failure”失败，这时虚拟机将启动后备预案：临时启用Serial Old收集器来 重新进行老年代的垃圾收集，这样停顿时间就很长了。所以说参数-XX：CM SInitiatingOccupancyFraction设置得太高很容易导致大量“Concurrent Mode Failure”失败，性能 反而降低。**
- CMS是一款基于“标记—清除”算法实现的收集器，这意味着收集结束时会有大量空间碎片产生。空间碎片过多时，将会给大对象分配带来很大麻烦，**往往会出现老年代还有很大空间剩余，但是无法找到足够大的连续空间来分配当前对象，不得不提前触发一次Full GC。**为了解决这个问题，CMS收集器提供了一个-XX：+UseCMSCompactAtFullCollection开 关参数（默认就是开启的），**用于在CMS收集器顶不住要进行FullGC时开启内存碎片的合并整理过程**，**内存整理的过程是无法并发的，空间碎片问题没有了，但停顿时间不得不变长。**



## JVM的内存分配策略

- 对象优先在Eden区分配:**大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。**
- **大对象直接进入老年代:所谓的大对象是指，需要大量连续内存空间的Java对象**，最典型的大对象就是那种很长 的字符串以及数组。大对象对虚拟机 的内存分配来说就是一个坏消息，经常出现大对象容易导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来“安置”它们。**虚拟机提供了一个-XX：PretenureSizeThreshold参数，令大于这个设置值的对象直接在老年代分配。** **这样做的目的是避免在Eden区及两个Survivor区之间发生大量的内存复制**。
- 既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代中。为了做到这点，虚拟机给每个对象定义了一个对 象年龄（Age）计数器。如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被 Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。**对象在Survivor区中每“熬过”一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁），就 将会被晋升到老年代中。**对象晋升老年代的年龄阈值，可以通过参数-XX： MaxTenuringThreshold设置
- 为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，**如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等 到MaxTenuringThreshold中要求的年龄。**
- **在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间**，如果这个条件成立，那么Minor GC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC。
- GC触发详解: https://www.jianshu.com/p/314272e6d35b



## 常用JVM监控工具



### jps: 虚拟机进程状况工具

除了名字像UNIX的ps命令之外，它的功能也和ps命令类似：可以列出 正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class,main（）函数所在的类）名称 以及这些进程的本地虚拟机唯一ID（Local Virtual Machine Identifier,LVMID）。虽然功能比较 单一，但它是使用频率最高的JDK命令行工具，因为其他的JDK工具大多需要输入它查询到 的LVMID来确定要监控的是哪一个虚拟机进程。对于本地虚拟机进程来说，LVMID与操作系统的进程ID（Process Identifier,PID）是一致的，使用Windows的任务管理器或者UNIX的ps命 令也可以查询到虚拟机进程的LVMID，但如果同时启动了多个虚拟机进程，无法根据进程名称定位时，那就只能依赖jps命令显示主类的功能才能区分了。jps可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，hostid为RMI注册表中 注册的主机名。



### jstat:虚拟机统计信息监视工具

jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据，在没有GUI图形界面，只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟 机性能问题的首选工具。

jstat的命令格式: 

```bash
jstat [option vmid [interval[s|ms] [count]]]
```



参数说明&常用选项如下图:

![image-20190927162512871](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162512871.png)



Example: 

![image-20190927162522435](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162522435.png)

- S0：幸存1区当前使用比例
- S1：幸存2区当前使用比例
- E：Eden区使用比例
- O：老年代使用比例
- M：元数据区使用比例
- CCS：压缩使用比例
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间



### jinfo:Java配置信息工具

jinfo（Configuration Info for Java）的作用是实时地查看和调整虚拟机各项参数。使用jps 命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参 数的系统默认值，除了去找资料外，就只能使用jinfo的-flag选项进行查询了（如果只限于 JDK 1.6或以上版本的话，使用java-XX：+PrintFlagsFinal查看参数默认值也是一个很好的选 择），jinfo还可以使用-sysprops选项把虚拟机进程的System.getProperties（）的内容打印出 来。这个命令在JDK 1.5时期已经随着Linux版的JDK发布，当时只提供了信息查询的功 能，JDK 1.6之后，jinfo在Windows和Linux平台都有提供，并且加入了运行期修改参数的能 力，可以使用-flag[+|-]name或者-flag name=value修改一部分运行期可写的虚拟机参数值。



### jmap:java内存映像工具

jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为heapdump或dump文 件）。如果不使用jmap命令，要想获取Java堆转储快照，还有一些比较“暴力”的手段：譬如 在第2章中用过的-XX：+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在OOM异常出 现之后自动生成dump文件，通过-XX：+HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break] 键让虚拟机生成dump文件，又或者在Linux系统下通过Kill-3命令发送进程退出信号“吓唬”一 下虚拟机，也能拿到dump文件。

jmap的作用并不仅仅是为了获取dump文件，它还可以查询finalize执行队列、Java堆和永 久代的详细信息，如空间使用率、当前用的是哪种收集器等。

![image-20190927162530621](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162530621.png)



###  jhat:虚拟机堆转储快照分析工具

Sun JDK提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆 转储快照。jhat内置了一个微型的HTTP/HTML服务器，生成dump文件的分析结果后，可以在 浏览器中查看。不过实事求是地说，在实际工作中，除非笔者手上真的没有别的工具可用， 否则一般都不会去直接使用jhat命令来分析dump文件，主要原因有二：一是一般不会在部署 应用程序的服务器上直接分析dump文件，即使可以这样做，也会尽量将dump文件复制到其 他机器 上进行分析，因为分析工作是一个耗时而且消耗硬件资源的过程，既然都要在其他 机器进行，就没有必要受到命令行工具的限制了；另一个原因是jhat的分析功能相对来说比 较简陋，后文将会介绍到的VisualVM，以及专业用于分析dump文件的Eclipse Memory Analyzer、IBM HeapAnalyzer 等工具，都能实现比jhat更强大更专业的分析功能。



###  jstack:Java堆栈跟踪工具

jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照（一般称为 threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈 的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循 环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。线程出现停顿 的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些 什么事情，或者等待着什么资源。



## Java内存模型

Java内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。此处的变量（Variables）与Java编程中所说的 变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但不包括局部变量与方法参数，因为后者是线程私有的 ，不会被共享，自然就不会存在竞争问题。为了获得较好的执行效能，Java内存模型并没有限制执行引擎使用处理器的特定寄存器或缓存来和主内 存进行交互，也没有限制即时编译器进行调整代码执行顺序这类优化措施。

**Java内存模型规定了所有的变量都存储在主内存（Main Memory）中**（此处的主内存与介绍物理硬件时的主内存名字一样，两者也可以互相类比，但此处仅是虚拟机内存的一部 分）。**每条线程还有自己的工作内存（Working Memory，可与前面讲的处理器高速缓存类比）**，线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝**(不会拷贝整个对象,可能是拷贝对象的引用或者对象在线程中用到的字段)** ，**线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的变量** 。 不同的线程之间也无法直接访问对方工作内存中的变量，**线程间变量值的传递均需要通过主内存来完成**

这里所讲的主内存、工作内存与Java内存区域中的Java堆、栈、方法区 等并不是同一个层次的内存划分，这两者基本上是没有关系的，如果两者一定要勉强对应起来，**那从变量、主内存、工作内存的定义来看，主内存主要对应于Java堆中的对象实例数据部分 ，而工作内存则对应于虚拟机栈中的部分区域。从更低层次上说，主内存就直接对应于物理硬件的内存，而为了获取更好的运行速度，虚拟机（甚至是硬件系统本身的优化措 施）可能会让工作内存优先存储于寄存器和高速缓存中，因为程序运行时主要访问读写的是工作内存。**

![image-20190927162537554](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162537554.png)

Java内存模型中定义了以下8种操作来 完成，虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的（对于double 和long类型的变量来说，load、store、read和write操作在某些平台上允许有例外)

- lock（锁定）：作用于主内存的变量，它把一个变量标识为一条线程独占的状态。

- unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内 存中，以便随后的load动作使用。
- load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工 作内存的变量副本中。
- use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引 擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作。
- assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内 存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存 中，以便随后的write操作使用。
- write（写入）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入 主内存的变量中。

**如果要把一个变量从主内存复制到工作内存，那就要顺序地执行read和load操作，如果要把变量从工作内存同步回主内存，就要顺序地执行store和write操作。**注意，Java内存模型**只要求上述两个操作必须按顺序执行，而没有保证是连续执行**。也就是说，read与load之 间、store与write之间是可插入其他指令的，如对主内存中的变量a、b进行访问时，一种可能 出现顺序是read a、read b、load b、load a。



## 原子性,可见性和有序性

Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这3个特征来建立的，我们 逐个来看一下哪些操作实现了这3个特性。

- 原子性（Atomicity）：由Java内存模型来直接保证的原子性变量操作包括read、load、 assign、use、store和write，我们大致可以认为基本数据类型的访问读写是具备原子性的（例 外就是long和double的非原子性协定，读者只要知道这件事情就可以了，无须太过在意这些几乎不会发生的例外情况）。如果应用场景需要一个更大范围的原子性保证（经常会遇到），**Java内存模型还提供了 lock和unlock操作来满足这种需求，尽管虚拟机未把lock和unlock操作直接开放给用户使用， 但是却提供了更高层次的字节码指令monitorenter和monitorexit来隐式地使用这两个操作，这 两个字节码指令反映到Java代码中就是同步块——synchronized关键字，因此在synchronized块 之间的操作也具备原子性。**

- 可见性（Visibility）：**可见性是指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。**上文在讲解volatile变量的时候我们已详细讨论过这一点。**Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作 为传递媒介的方式来实现可见性的**，无论是普通变量还是volatile变量都是如此，**普通变量与volatile变量的区别是，volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。**因此,可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。**除了volatile之外，Java还有两个关键字能实现可见性，即synchronized和final。**同步块的可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、 write操作）”这条规则获得的，而final关键字的可见性是指：被final修饰的字段在构造器中一旦初始化完成，并且构造器没有把“this”的引用传递出去（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到“初始化了一半”的对象），那在其他线程中就能看见final字段的值。**(final变量的赋值操作可以保证可见性)**

- 有序性（Ordering）：Java内存模型的有序性在前面讲解volatile时也详细地讨论过 了，Java程序中天然的有序性可以总结为一句话：**如果在本线程内观察，所有的操作都是有序的；如果在一个线程中观察另一个线程，所有的操作都是无序的。**前半句是指“线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics），后半句是指“指令重排序”现象 和“工作内存与主内存同步延迟”现象。 Java语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性，**volatile 关键字本身就包含了禁止指令重排序的语义，而synchronized则是由“一个变量在同一个时刻 只允许一条线程对其进行lock操作”这条规则获得的**，这条规则决定了**持有同一个锁的两个同步块只能串行地进入。**



## JVM的锁优化

### 自旋锁与自适应自旋锁

**互斥同步对性能最大的影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给系统的并发性能带来了很大的 压力。**同时，虚拟机的开发团队也注意到在许多应用上，共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程并不值得。如果物理机器有一个以上的处理器，能让两个或以上的线程同时并行执行，**我们就可以让后面请求锁的那个线程“稍等一 下”，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。为了让线程等待，我们只需让线程执行一个忙循环（自旋），这项技术就是所谓的自旋锁。**

自旋锁在JDK 1.4.2中就已经引入，只不过默认是关闭的，可以使用-XX：+UseSpinning 参数来开启，在JDK 1.6中就已经改为默认开启了。自旋等待不能代替阻塞，且先不说对处理器数量的要求，自旋等待本身虽然避免了线程切换的开销，但它是要占用处理器时间的， **因此，如果锁被占用的时间很短，自旋等待的效果就会非常好，反之，如果锁被占用的时间 很长，那么自旋的线程只会白白消耗处理器资源，而不会做任何有用的工作，反而会带来性 能上的浪费。因此，自旋等待的时间必须要有一定的限度**，如果自旋超过了限定的次数仍然 没有成功获得锁，就应当使用传统的方式去挂起线程了。自旋次数的默认值是10次，用户可 以使用参数-XX：PreBlockSpin来更改。

**在JDK 1.6中引入了自适应的自旋锁**。自适应意味着自旋的时间不再固定了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有 可能再次成功，进而它将允许自旋等待持续相对更长的时间，比如100个循环。另外，如果对于某个锁，自旋很少成功获得过，那在以后要获取这个锁时将可能省略掉自旋过程，以避免浪费处理器资源。有了自适应自旋，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测就会越来越准确，虚拟机就会变得越来越“聪明”了。

### 锁消除

锁消除是指**虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。**锁消除的主要判定依据来源于逃逸分析的数据支持，**如果判断在一段代码中，堆上的所有数据都不会逃逸出去从而被其他线程访问到，那就可以把它们当做栈上数据对待，认为它们是线程私有的，同步加锁自然就无须进行。**

### 锁粗化

我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小——只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小，如果存在锁竞争，那等待锁的线程也能尽快拿到锁。但是如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。

### 轻量级锁

轻量级锁是JDK 1.6之中加入的新型锁机制，它名字中的“轻量级”是相对于使用操作系统 互斥量来实现的传统锁而言的，因此传统的锁机制就称为“重量级”锁。首先需要强调一点的 是，轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

### 偏向锁

Java偏向锁(Biased Locking)是Java6引入的一项多线程优化。 

偏向锁，顾名思义，它会偏向于第一个访问锁的线程，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要触发同步的，减少加锁／解锁的一些CAS操作（比如等待队列的一些CAS操作），这种情况下，**就会给线程加一个偏向锁。 如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁。**

### 三种锁的区别

![image-20190927162549813](/Users/nacht/Documents/Talking-To-The-Moon/tech_notes/java/assets/image-20190927162549813.png)

- **偏向锁: 仅仅有一个线程进入临界区**
- **轻量级锁: 多个线程交替进入临界区**
- **重量级锁: 多个线程同时进入临界区**