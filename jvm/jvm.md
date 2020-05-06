### 内存模型
- 分为：5个
    - 程序计数器，
    - 虚拟机栈
       每个方法被执行的时候都会创建一个栈帧Stack-Frame，完整的栈帧包括局部变量表、操作数栈、动态连接信息、方法正常完成和异常完成信息
    - 本地方法区，
       本地方法栈则是为虚拟机使用到的Native方法服务。例如java与mysql交互，底层就是jvm调用自己的本地方法接口，JNI.
    - 堆（新生代，老生代），
    - 方法区（方法区被称为永久代,内含运行常量池）
       与堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据
      
### 垃圾回收算法
- 1.标记清除
    1.标记阶段首先通过根节点(GC Roots)，标记所有从根节点开始的对象，未被标记的对象就是未被引用的垃圾对象。
    2.清除阶段，清除所有未被标记的对象
    3.存活对象较多的情况下比较高效
- 2.复制算法
    1.从根集合节点进行扫描，标记出所有的存活对象，并将这些存活的对象复制到一块儿新的内存
- 3.标记整理
     先标记，后压缩，最后清除
- 4.分代收集算法
  - 在不同年代使用不同的算法，从而使用最合适的算法
     - 新生代存活率低，可以使用复制算法。   Minor GC方式，速度快、效率高的
         新生代内存按照8:1:1的比例分为一个eden区和两个survivor(survivor0,survivor1)区
     - 老年代对象存活率搞，没有额外空间对对象进行分配担保，所以只能使用标记清除或者标记整理算法。 Full GC
           
### 垃圾回收器
- 年轻代的垃圾回收器：
   - serial 单线程收集器， 单线程收集器，在进行垃圾收集时，必须暂停其他所有的工作线程
   - parNew 是serial的多线程版本，  
   - parallel-scavenge 是用在计算等吞吐量应用上的第一种选择
- 老年代的回收器
   - serial-old 是老年代的单线程收集器
   - cms ： 初始标记、并发标记、重新标记、并发清除
   - parallel-old是parallel-scavenge 在老年代的中版本，他俩经常一起使用。
   - g1 垃圾收集器
      - 和前代收集器相似点：
         1.G1有类似CMS的收集动作：初始标记、并发标记、重新标记、清除、转移回收，
         2.并且也以一个串行收集器做担保机制
      - 不同点：
         1.设计原则是"首先收集尽可能多的垃圾(Garbage First)"
           1.1 不会等内存耗尽时候开始垃圾收集.而是在内部采用了启发式算法，在老年代找出具有高收集收益的分区进行收集。
           1.2 可以根据用户设置的暂停时间目标自动调整年轻代和总堆大小，暂停目标越短年轻代空间越小、总空间就越大
         2.G1采用内存分区(Region)的思路，将内存划分为一个个相等大小的内存分区，回收时则以分区为单位进行回收
         3.G1虽然也是分代收集器，但整个内存分区不存在物理上的年轻代与老年代的区别，只有逻辑上的分代概念，每个分区都可能随G1的运行在不同代之间前后切换
         4.G1的收集都是STW的，但年轻代和老年代的收集界限比较模糊，采用了混合(mixed)收集的方式
- cms垃圾回收器 详解
   - 优点：并发收集、低停顿。 
   - 缺点：
       1.CPU资源敏感。 
          CMS在收集与应用线程会同时会增加对堆内存的占用.CMS必须要在老年代堆内存用尽之前完成垃圾回收；
          否则CMS回收失败时，将触发担保机制，串行老年代收集器将会以STW的方式进行一次GC，从而造成较大停顿时间
       2.无法处理浮动垃圾（Floating Garbage），即无法收集并发运行中产生的新的垃圾。 
       3.容易产生空间碎片。为了解决这个问题，
           - +UserCmsCompactAtFullCollection（default=on）这个过程是无法并发的，所以每隔一段时间就会出现停顿时间稍长的问题。
           - +XX:CMSFullGCsBeforeCompaction，（default=0，每次Full GC时都进行碎片整理）。


### 如何判断一个对象是否可以被回收？
- 引用计数法  很难解决对象之间的循环引用问题
- 枚举根节点做可达性分析
    通过一系列名为“GC Roots”的对象作为起始点，从“GC Roots”对象开始向下搜索，
    如果一个对象到“GC Roots”没有任何引用链相连，说明此对象可以被回收。

### 哪些对象可以作为 GC Roots 的对象：
* 虚拟机栈中局部变量（也叫局部变量表）中引用的对象
* 方法区中类的静态变量、常量引用的对象
* 本地方法栈中 JNI (Native方法)引用的对象 

### 触发fullGC的条件
- 1.System.gc  
      可能触发fullGc
- 2.旧生代空间不足
      旧生代空间只有在新生代对象转入及创建大对象、大数组时才会出现不足的现象，
- 3.JDK 1.7 及以前的永久代空间不足
- 4.CMS GC时出现promotion failed和concurrent mode failure
      - promotionfailed是在进行Minor GC时，survivor space放不下、对象只能放入旧生代，而此时旧生代也放不下造成的；
      - concurrent mode failure是在执行CMS GC的过程中同时有对象要放入旧生代，而此时旧生代空间不足造成的。
- 5.空间分配担保机制. (失败时)
     在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间。
如果这个条件成立，那么Minor GC可以确保是安全的。
如果不成立，则虚拟机会查看HandlerPromotionFailure设置是否允许担保失败。
如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小。
如果大于，将尝试着进行一次Monitor GC，尽管这次GC是有风险的。
如果小于，或者HandlerPromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC了。