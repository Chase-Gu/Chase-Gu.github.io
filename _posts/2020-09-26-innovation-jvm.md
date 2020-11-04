---
layout:     post
title:      「日记」暑期JVM大创总结
subtitle:   通过JVMTI监控JVM和日志分析实现GC可视化
date:       2020-09-26 00:09
update:     2020-09-30 16:48
author:     Chase Gu
header-img: img/home-bg-o.jpg
hide: false
catalog:    false
my-keys:    jvmtijvm大创gc可视化
tags:
    - 日记
    - 大创
    - Java
    - JVM
---

临近开学的时候就准备写一篇总结，结果没想到暑假已经结束了快一个月了才有时间写哈哈哈。

Java虚拟机GC可视化，第一次拿到这个题目，感觉应该不是很难，毕竟对我来说Java算是最熟悉的一门语言了。但是仔细一想便发现事情并没有那么简单，甚至无从下手。项目的理想需求是实现JVM的GC可视化。当前已有的工具基本上都只提供了一个统计的数据，例如jconsole，只能提供堆的占用量，并不能提供精确到对象的堆的信息。所以本次项目的目的就是要实现更细粒度的可视化，它不仅能够显示对象的位置——是新生代还是老生代？是伊甸区还是Survivor区？还能在GC时动态地显示什么对象被释放了，当然，已有的统计的数据也要求能显示。

可视化的前提是要有充足和准确的数据。对于高度封装的JVM，要想获取运行期间的数据的确不是一件容易的事情。如果你是一个C++开发人员，你可以通过`&`运算符轻而易举地获取到变量的地址，但是在Java，你可能需要通过反射来创建Unsafe类的对象，然后通过复杂的逻辑处理才能得到地址。更别说要动态地获取到对象在堆中的分代、分区信息了。

于是我想，不是还有GC日志吗，实在不行就可视化GC日志算了。当然这是说笑的，就GC日志那点东西还不够塞牙缝的。但是GC日志给了我一个想法：是不是还有一些我们不知道的有关GC的JVM参数，说不定组合在一起，再通过逻辑推理能够得到预期结果呢？抱着这个想法，我决定先开始看书，于是我们小组四个人开始看周志明老师编写的《深入理解Java虚拟机》，因为我认为，只有首先拓展知识面，然后才能根据自己的理解，综合使用各种方法，最终实现目标。

在读书的同时，我们还看了一点点点Hot Spot虚拟机的源代码。为什么看虚拟机源码呢？这主要是因为最开始写项目申报书的时候，我们搜到了一篇可视化Java虚拟机的论文。作者通过定制修改虚拟机来实现了一定程度上的Java虚拟机的可视化。不过遗憾的是，在GC方面提供的可视化信息并不多。也正是因为这篇论文，再加上一开始我们对JVM理解很浅显，知识还不足，所以我们当时想：也许只有像他一样魔改虚拟机才能实现目标吧！于是一位组员（Alex）下载了一份open jdk8的源代码，并分享给我，我们希望能够找到Hot Spot分配对象部分以及堆中分代分区部分的代码，然后植入业务代码，最后编译成为修改后的虚拟机来实现任务。为此Alex还在Linux上成功编译了open jdk8的源代码。然而遗憾的是，我们并没有找到控制堆中分代分区的代码，只有一些新生代老生代的字眼，分析之后又跳到了不知道干什么的代码上。想要理解代码，可能需要通读所有的Hot Spot源码，这在只有源代码的情况下几乎是不可能的，并且我们寻找了很多视频和书籍，也都没有专门针对源码的解读，没有整体结构的介绍。这使得我们不得不思考，是不是还有别的方法？

庆幸的是，在两周的准备之后，事情迎来了转机。在最初的两周里，我们阅读了《深入理解Java虚拟机》，搜索了包括GC的JVM参数、MXBean、Unsafe、Instrumentation插桩等等各种各样的可能有帮助的线索，最终我将目光锁定在了JVMTI上。因为各类博客对JVMTI的介绍是“Java Vitual Machine Tool Interfaces——JVM提供的一套工具接口”，并且可以明确指出了可以监控GC。但是当我搜索JVMTI的教程的时候，网络上几乎所有的博客都是转载的同一篇博文，而且这篇博文一点也不系统，上来就是满口的agent、attach API这种“高深”的东西，一套“之乎者也”把还不知道JVMTI怎么用的我打得脑袋瓜子嗡嗡的。最后我无意间找到了<a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html" target="_blank">JVMTI官方文档</a>，事情才算有了进展（所以说**看博文也就图一乐，真要学习还得看官方文档**），其中的<a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#IterateThroughHeap" target="_blank">`IterateThroughHeap`</a>方法引起了我的注意，以下是官方文档的介绍：

> Initiate an iteration over all objects in the heap. This includes both reachable and unreachable objects. Objects are visited in no particular order.
>
> Heap objects are reported by invoking the agent supplied callback function <a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#jvmtiHeapIterationCallback" target="_blank">`jvmtiHeapIterationCallback`</a>. References between objects are not reported. If only reachable objects are desired, or if object reference information is needed, use <a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#FollowReferences" target="_blank">`FollowReferences`</a>.

对堆进行迭代、仅仅迭代被引用到的对象……这难道不正是我想要的吗？也正是从这个时候开始，整个项目后端部分的研究重点转移到了JVMTI上。

通过阅读文档，我花费了几天的时间来理解JVMTI的运作原理。根据我个人的理解，JVMTI是JVM提供的一套接口，而使用接口的方式是通过C/C++语言编写一个agent，然后连接上JVM。JVM在运行时会触发不同的<a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#EventSection" target="_blank">事件（Event）</a>，然后调用事件对应的回调函数，在回调函数中我们可以使用JVMTI和JNI函数或者自己编写的业务函数（特殊情况除外）。例如`VMInit`即在虚拟机初始化完成之后调用回调函数，我们可以在回调函数中使用<a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#IterateThroughHeap" target="_blank">`IterateThroughHeap`</a>，然后编写`IterateThroughHeap`的回调函数，累加迭代得到的对象的大小，从而获取虚拟机初始化完成之后堆的使用量。

在理解了JVMTI的运作原理之后，我们开始了初步的尝试。当时搜索到的编译方式，只有在Linux系统上能够成功编译，所以我们只得在没有提示的情况下，在Windows上编写好agent程序，然后拷贝到卡的要死的Ubuntu上测试。由于没有提示，编译的时候错误连篇，直接在Ubuntu上编写又卡的飞起，简直让人心态爆炸。最后就成了主要我在Windows上编写，然后发给Alex让他在Ubuntu上测试。在这样低效率的环境下，我们编写了第一个成功的Agent——在`VMInit`事件中调用`iterateThroughHeap`函数。

看到自己对JVMTI运行原理的理解是正确的之后，我们便开始按照项目需求编写agent。最初我们的策略是在`GarbageCollectionStart`事件中调用`FollowReferences`函数来获取堆中所有能够被引用到的对象。然而在长达一个多星期的各种失败尝试，各种`fatal error`之后，我们发现忽略了文档中的一句话：

> A Garbage Collection Start event is sent when a garbage collection pause begins. Only stop-the-world collections are reported--that is, collections during which all threads cease to modify the state of the Java virtual machine. This means that some collectors will never generate these events. **This event is sent while the VM is still stopped, thus the event handler must not use JNI functions and must not use JVM TI functions except those which specifically allow such use (see the raw monitor, memory management, and environment local storage functions).**

也就是说由于GC时，VM是停止的，所以不能使用JNI函数和JVMTI函数。同时也提出了一个新名词——`raw monitor`，莫非是要使用多线程？

在上面的agent的编写过程中，我还搜索到，open jdk8的demo文件夹下有一个jvmti文件夹，其中有一些官方提供的jvmti demo（`openjdk-8u41-src-b04-14_jan_2020\openjdk\jdk\test\demo\jvmti`）。我告诉了Alex，随后Alex借鉴了其中的Worker方式，使用`raw monitor`在`GarbageCollectionStart`事件的时候通过另一个线程来执行`FollowReference`函数。然而，噩耗又来了。在当我们觉得这个思路无懈可击的时候，JVM它又抛出了`fatal error`，这个`fatal error`卡住了我们的进度，一卡就是一个多星期。

这一切，在一次偶然发现之后迎来了转机。一天，我尝试在Windows下编译agent，结果发现之前编译失败的原因居然是目录名存在空格（这个电脑买来目录就那个怪样子，我没注意）。解决了Windows编译问题之后，我们又自定义了VS Code的tasks，实现一键编译。并且出人意料的是，这个玩意居然是平台相关的。我们将在Ubuntu上出错的、使用worker方式的程序在Windows上重新编译运行，居然能够正常运行！自此，我们的效率提升了几个数量级。

在接下来的开发，大体经历了五个阶段：

- TIClassLoadTag阶段
- TIClassLoadTag和TIGetLoaded阶段
- Tracker初级阶段
- Tracker+jmap构想
- Tracker+Analyzer阶段

**TIClassLoadTag阶段**

既然已经能够遍历堆中对象，那么是时候想想怎么获取到对象的信息了。`IterateThroughHeap`的回调函数（`jvmtiHeapIterationCallback`）包含一个`class_tag`参数，这表示对象的类的标签。所以只要在合适的时候为类打上`tag`，就能够在遍历的时候知道对象的类型了。于是我们在`ClassPrepare`事件的回调函数中调用JVMTI函数`SetTag()`为类打上唯一标识`tag`。但是令人失望的时候，这个时候堆迭代中会出现`fatal error`，并且经过问题排查发现只要去掉`ClassPrepare`事件，保证该事件回调函数不被触发就不会出现`fatal error`。这个问题至今还没有得到答案，我猜测是因为堆迭代期间可能有类的加载，触发了`ClassPrepare`事件导致迭代异常。

**TIGetLoaded阶段**

如果不能够触发`ClassPrepare`，那么其他事件是否可以呢？经过我的测试，`ClassLoad`、`ClassPrepare`以及`ClassFileLoadHook`事件与GC类事件回调函数均不能共存。既然不能在类加载或加载完成等阶段调用回调函数，那就再其他事件中设置`tag`。于是我又在`VMinit`事件回调中调用了`GetLoadedClasses`方法获取到已经加载的类，并依次调用`SetTag()`，但是很明显这样有一个问题，类的加载是一个动态的过程，不可能`VMInit`之后就不会再变化了，所以这种方法还是有很多类没有被标记上的。

**Tracker阶段**

做到这里，进行堆迭代的路似乎已经走到头了，这不得不迫使我重新考虑之前有过的设想：如果对象创建能够触发回调函数，就可以直接对对象设置标签，同时也可以调用`GetClassSignature()`。但是在<a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html" target="_blank">JVMTI官方文档</a>中有这样一句话让当时的我们放弃了这项方案：

> Sent when a method causes the virtual machine to allocate an Object visible to Java programming language code and the allocation is not detectable by other intrumentation mechanisms. Generally object allocation should be detected by instrumenting the bytecodes of allocating methods. Object allocation generated in native code by JNI function calls should be detected using <a href="https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#jniIntercept" target="_blank">JNI function interception</a>. Some methods might not have associated bytecodes and are not native methods, they instead are executed directly by the VM. These methods should send this event. Virtual machines which are incapable of bytecode instrumentation for some or all of their methods can send this event.
>
> Typical examples where this event might be sent:
>
> - Reflection -- for example, `java.lang.Class.newInstance()`
> - Methods not represented by bytecodes -- for example, VM intrinsics and J2ME preloaded classes
>
> Cases where this event would not be generated:
>
> - Allocation due to bytecodes -- for example, the `new` and `newarray` VM instructions
> - Allocation due to JNI function calls -- for example, `AllocObject`
> - Allocations during VM initialization
> - VM internal objects

这意味着，普通的new语句、JNI的创建对象函数都不会触发`VMObjectAlloc`事件，这种情况下的对象创建应该被字节码增强机制（intrumentation mechanisms）或者JNI的拦截所检测到。

但是一次偶然的机会，我找到了这个提问：<a href="https://www.it1352.com/1815837.html" target="_blank">受jvmti对象分配回调行为困扰(Perplexed by jvmti object allocation callback behavior)</a>，开辟了一条全新的道路。

> For performance reasons, the JVMTI only supports allocation events for objects that cannot be detected through bytecode instrumentation (BCI), as explained in the JVMTI VMObjectAlloc event documentation. This means that the event will not be triggered for most object allocations. I assume that the allocations you are missing fall within that category.
>
> Fortunately, it's not really too difficult to intercept all object allocations using BCI. The [HeapTracker](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/tip/src/share/demo/jvmti/heapTracker/) demo illustrates precisely how to intercept all object allocations in a JVMTI agent using [java_crw_demo](https://blogs.oracle.com/kto/entry/a_short_java_crw_demo) in addition to the VMObjectAlloc events.
>
> 出于性能原因， JVMTI仅支持无法通过字节码检测（BCI）检测到的对象的分配事件，如JVMTI VMObjectAlloc 事件文档中所述。这意味着该事件不会为大多数对象分配触发。我假设您缺少的分配属于该类别。
>
> 幸运的是，使用BCI拦截所有对象分配并不是真的很困难。 HeapTracker 该演示精确地说明了如何使用 java_crw_demo 拦截JVMTI代理中的所有对象分配。 VMObjectAlloc 事件。

提问者提问的是为什么new为什么不触发`VMObjectAlloc`事件，解答者指明了一个方向：HeapTracker演示中有拦截JVMTI代理中所有对象分配的方法。HeapTracker是open jdk8的demo目录的jvmti目录下的一个演示文件（`openjdk-8u41-src-b04-14_jan_2020\openjdk\jdk\test\demo\jvmti`）。于是我们花了几天时间来理解HeapTracker的运作原理：代码实在是太长了。

受到HeapTracker的启发，我们决定采用这种方案：首先将需要追踪的class文件打jar包，在Agent onLoad的时候将jar包加入（调用`AddToBootstrapClassLoaderSearch`方法）；随后创建本地方法，接受Java对象作为参数，并在`VMInit`事件回调中注册本地方法。同时，Java代码中需要在创建对象时将对象传入本地方法，这样就可以在Agent（我们的JVMTI程序）中获取到Java对象。

我们使用Javassist工具对class文件进行插桩，这部分工作主要由Alex完成，详细过程可见他的博客：<a href="http://alexande.gitee.io/blog/2020/09/24/javassist/" target="_blank">字节码增强与Javassist工具库的使用（一）</a>和<a href="http://alexande.gitee.io/blog/2020/09/26/javassist2/" target="_blank">字节码增强与Javassist工具库的使用（二）</a>。创建本地静态方法，遍历方法中的字节码，找到创建实例的指令，然后调整操作栈顶的数据，使得可以调用native函数。

在Agent中，我设置了三个映射（Map）分别用于维护`tag`到TObject对象的映射、`classTag`到类的`signature`的映射以及类的`signature`到`classTag`的映射。

```c++
map<long long, TObject*> tObjectMap;    // Save [objectTag, TObject]
map<int, string>         classMap;      // Save [classTag, signature]
map<string, int>         classTagMap;   // Save [signature, classTag]
```

Agent获取到Java对象之后，首先通过`GetObjectClass`和`GetClassSignature`函数获取到对象的类签名，并获取对象的大小，将这些信息封装成自己创建的`TObject`类中。接着调用`SetTag`为对象打上标签，并加入到`tObjectMap`映射中。然后，该类包含对象的类标签`classTag`，对象的大小，对象名等信息。然后将该类加入到Map中，自己维护一个`classTag`与`signature`的双向映射——这样既可以在`VMObjectAlloc`的时候得知该对象对应的类是否已经记录，并且返回类对应的`classTag`从而记录到对象对应的`TObject`对象中，也可以在对象释放的时候通过对象的`tag`获取到对象的类的`classTag`从而得知`signiture`。

在`VMObjectFree`事件回调中，能够获取到被释放对象上的`tag`，然后通过`tObjectMap`找到该`tag`对应的`TObject`，从而获取基本信息，同时在通过`TObject`存储的`classTag`，查询`classMap`获取到被释放对象的类签名。

自此我们已经能够准确地监控对象的分配与释放，准确地获取到对象的大小、类签名信息。但是我们仍不知道对象的具体位置——堆中的代和区的位置，并且JVMTI也没有提供任何有关分代分区的接口。于是我决定通过这些信息进行反推。我编写了一个`THeap`类用于模拟虚拟机的标记复制和标记整理算法，计算堆中各代的占用量，从而分析出对象创建的具体位置。

之后，在我们的测试中，我们发现在虚拟机初始化完成`VMInit`之前回创建很多虚拟机需要的对象，这些对象也占用了堆的空间。于是我有添加了一个`VMInit`回调事件，在其中调用`IterateThroughHeap`来累加获取虚拟机初始化完成之后的堆占用量。将这个占用量来初始化`THeap`类，从而达到更为精确的分析。

**Tacker+jmap构想**

得到了第一个版本的Tracker，我们喜出望外，但是很快便意识了问题。虚拟机创建的对象，我们并没有办法为其打上标签，所以它们的分配和释放时完全在我们的监控之外的。我们只能够在`VMInit`的时候得知它们的总大小，但是在一次次GC之后，它们之中有哪些释放了，有哪些存活了，有哪些晋升到老年代了，从理论上我们就无法得知。这就意味着，我们现在的程序会随着GC次数的增多变得越来越不准确。

为了解决这个问题，我想到了以前看《深入理解Java虚拟机》的时候看到的jmap指令。jmap能够获取到堆中各分代的容量和占用量大小。或许我们能够通过jmap的信息来校准`THeap`的分析呢？于是我在`GCFinish`事件中执行了jmap指令，但是在各种尝试之后，不是`fatal error`就是死锁。

根据查找的资料和我个人的理解，jmap是建立与JVM的连接，然后向JVM发送指令，JVM执行完之后再把结果返回。所以如果JVM无法继续执行，jmap应该是得不到结果的。而我们的Agent是由JVM调用的，如果Agent在等待jmap结果，jmap又在等待JVM执行，那么将会发生死锁。

在此之前，所有的方案都是在Agent中一边执行一遍模拟，所以可能会发生死锁。那么不妨将模拟分析独立成一个进程，Agent在必要的时候通知分析进程并发送一些必要的信息，如对象分配，随后分析进程再调用jmap指令，利用二者的信息来实现分析。但是这个想法只停留在构想，并没有实现。一方面我以前没有写过进程间通信 (TvT)，另一方面是可能存在误差，消息传递不够及时。

**Tracker+Analyzer阶段**

虽然Tracker+jmap的构想并没有付诸行动，但是将分析模块独立出来的想法提供了一个新的突破口。我想，不如将分析模块独立得更彻底一些。于是在这个方案中，我们完全移除了`THeap`类，不再在JVMTI Agent中执行分析操作，Agent只专心获取信息，并将信息写入文件。同时通过`-XX:+PrintGCDetails`参数输出GC日志，因为GC日志也是可以提供堆中各代的信息的。分析程序读取这两份文件，通过Agent的信息模拟堆的分配释放与垃圾回收，并在GC的时候通过GC日志对模拟的信息进行校准。这样就可以通过校准来排除JVM创建的对象的影响。

在编写好了JVMTI Agent之后，我们开始编写Analyzer程序。我们使用kotlin编写了一个Analyzer，主要分为三个层次。其中最底层读取两份日志并形成Event对象列表，中层依次遍历Event对象列表模拟堆的操作，并执行上层的回调函数，上层通过中层的模拟来形成Bean对象列表（这怪名字Alex起的）供前端显示使用。

此时，大创小组的前端同学们都在学习D3.js，我想可能光D3.js并不够，于是在编写了一部分Analyzer之后，开始了一个星期的学习开发时间。这期间我一边看HTML、css和JavaScript，一边通过编写可视化部分试一试，最后写出了一个“表面风平浪静，实则底层暗流涌动”的“屎山”。虽然实现了信息的可视化，并且页面也还凑合，但是由于是使用原生、没有使用任何框架，代码耦合度非常高，所以说前端小组前端估摸着还是得重写啊哈哈哈。同时，这段时间的工作也不是没有意义的。在编写可视化的过程中，我们也发现了一些Analyzer中没有考虑到的问题，所以这段时间也是一段优化Analyzer的时期。

做到这里，暑假也快要结束了！对我而言，收获的不仅仅是虚拟机的运行原理，或者JVMTI Agent的开发方法。最重要的是从毫无思路开始，一点一点地寻找方法，并且思考各种方案来解决问题的过程。初次之外，如何进行团队的管理也是一项不小的收获。其实在大一的时候，我就参加过一个大创，不过那次负责人不是我。那次大创负责人没有安排好工作，导致很长时间都没有人做，以至于之后如果有组员想做，发现没有其他一个组员做，找负责人也没有回应，就不想做了。每个组员都这么想，自然也就没有做，最后就不了了之了。所以这一次我吸取了教训，自己做了负责人。在暑假期间，因为缺乏监督，为了防止划水，我决定每周开一次组会，每周写周报，叙述自己的进展和接下来的计划并上交，来起到一个监督和交流的作用。于是，在我所知的大创小组中，我们组是进展最早、进展最快的一组，从事实证明了团队管理的重要性。

—— 2020年09月26日0时 至 2020年09月30日16时