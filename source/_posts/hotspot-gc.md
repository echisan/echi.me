---
title: hotspot垃圾收集器
date: 2019-11-29 18:08:15
tags:
  - gc
  - hotspot
  - java
  - jvm
---

## hotspot虚拟机垃圾收集器

垃圾收集器目前来说有：
- Serial(串行)
- parnew(Serial收集器的多线程版本)
- parallel scavenge(并行清除)
- serial old
- parallel old
- cms
- g1

**新生代收集器**
- Serial
- ParNew
- Parallel Scavenge
- (G1)

**老年代收集器**
- CMS
- Serial Old(MSC)
- Parallel Old
- (G1)

## 复习下收集算法

### 标记-清除算法

描述：如同名字一样，先标记后收集
缺点： 效率问题，标记、收集两个过程效率都不高。另一个是空间问题，标记清除后会产生大量不连续的内存碎片，导致以后程序需要分配较大对象时，无法找到足够内存而不得不触发另一次垃圾收集动作

![image](https://user-images.githubusercontent.com/38010908/69610588-0e8cb600-1067-11ea-9a76-95530c6c723d.png)

### 复制算法

描述：将可用内存按照容量分为大小相等的两块，当这一块内存用完了，就将还存活的对象复制到另一块上，然后再把已使用过的内存空间一次清理掉。
优点：进行垃圾收集时，对整个半区进行内存回收，内存分配也不用考虑内存碎片等复杂情况
缺点：将内存缩小了为原来的一半，代价过大

![image](https://user-images.githubusercontent.com/38010908/69611524-e8681580-1068-11ea-8ac5-7130372ce75f.png)

> 现在的商业虚拟机都采用该收集算法来回收新生代，由于新生代中对象98%是“朝生夕死”的，所以不需要1:1比例划分内存空间，所以分为一块较大的Eden空间以及两块较小的Survivor空间，每次使用eden以及其中一块Survivor

回收过程：  
将Eden以及Survivor中还存活的对象一次性复制到另外一块Survivor空间中，最后清理掉刚才用过的Survivor空间

> 我们没办法确保每次回收都只有不多于10%的对象存活，当Survivor空间不够用时，需要依赖其他内存(这里指老年代)进行分配担保。

分配担保：
> 在垃圾收集过程中，如果另外一块Survivor没有足够空间存放上一次新生代收集下来的对象，则通过分配担保机制进入老年代。


担保机制：  

主要讲JDK1.6之后的。  
如果老年代可用的连续内存大小大于新生代对象总大小或者对象历次晋升的平均大小(也就是每次经过MinorGC之后还存活的对象的平均大小)，那么就进行MinorGC，否则就进行Full GC。


![image](https://user-images.githubusercontent.com/38010908/69612170-244faa80-106a-11ea-98e9-3e16ae7f0e9d.png)


## 标记-整理算法
复制-收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中对象100%存活的极端情况，所以老年代一般不能直接选用这种算法。

根据老年代的特点，标记过程仍然与(标记-清除)一样，但是清除步骤换成整理，让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存

![image](https://user-images.githubusercontent.com/38010908/69615011-210aed80-106f-11ea-984d-dce14368fa19.png)

## 分代收集算法

当代商业虚拟机的垃圾收集都采用“分代收集”(Generational Collection)算法，这种算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。  
一般是把java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。

---

## Parallel Scavenge
并行的多线程的收集器，看似跟ParNew是一样的，但是是有区别的。

> Parallel Scavenge收集器架构中本身有PS MarkSweep收集器进行老年代收集，并非直接使用Serial Old收集器，但是这个PS MarkSweep收集器与Serial Old的实现非常接近

Parallel Scavenge收集器的目标是达到一个可控制的吞吐量。  
所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即

`吞吐量 = 运行用户代码时间 / (运行用户代码时间+垃圾收集时间)`

Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量
- 控制最大垃圾收集器停顿时间(-XX:MaxGCPauseMills)
  > 允许值是一个大于0的毫秒数
- 设置吞吐量大小(-XX:GCTimeRatio)
  > 参数是一个大于0且小于100的整数，也就是垃圾收集时间占总时间比率，相当于是吞吐量的倒数
- 自适应调节策略(-XX:+UseAdaptiveSizePolicy)
  > 打开这个参数后不需要手动指定新生代的大小(-Xmn)、Eden与Survivor区的比例(-XX:SurvivorRatio)、晋升老年代对象年龄(-XX:PretenureSizeThreshold)等参数，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适停顿时间或最大的吞吐量

## Parallel Old
Parallel Old 是Parallel Scavenge Scavenge收集器的老年代版本，使用多线程和“标记-整理"算法

> 在没有Parallel Old收集器的时候，如果选择了Parallel Scavenge作为新生代收集器，那么老年代出了Serial Old收集器之外别无选择

直到Parallel Old收集器出现后，“吞吐量优先”的收集器才有了比较名副其实的应用组合，在注重吞吐量以及CPU资源敏感的场合，可以优先考虑Parallel Scavenge加Parallel Old收集器

![image](https://user-images.githubusercontent.com/38010908/69698111-62f36c80-111f-11ea-92b1-a53870f69d4c.png)


## Serial && Serial Old
单线程  
Serial新生代使用复制算法  
Serial old老年代使用标记整理算法

> Serial Old收集器意义在于给client模式下的虚拟机使用。  
> 在Server模式下，有两大用途：1、在JDK1.5以及之前版本与Parallel Scavenge收集器配合使用，另一种就是作为CMS收集器后备预案，在并发收集发生Concurrent Mode Failure时使用 

![image](https://user-images.githubusercontent.com/38010908/69610367-a047f380-1066-11ea-8280-20eb52c5ccb6.png)


## ParNew

ParNew收集器其实就是Serial收集器的多线程版本，是Server模式下的首选新生代收集器(主要是因为只有ParNew收集器能与CMS老年代收集器配合工作)

> 不幸的是，CMS无法与Parallel Scavenge新生代收集器配合使用。  
> 所以如果选择CMS作为老年代收集器，新生代只能选择ParNew跟Serial收集器其中一个

所以使用`-XX:+UseConcMarkSweepGc`的选项后ParNew也是默认的新生代收集器

> 可以使用`-XX:ParallelGCThreads`参数限制垃圾收集的线程数

![image](https://user-images.githubusercontent.com/38010908/69622727-7bf71180-107c-11ea-88a1-cb515a1af4fe.png)


## CMS
CMS(Concurrent Mark Sweep)收集器是一种以获取最短回收停顿时间为目标的收集器。

从名字Mark Sweep可以看出来，CMS也是基于标记-清除算法实现的，但是比前面几种收集器来说更复杂一点。
- 初始标记(CMS initial mark)
  > 初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快 
- 并发标记(CMS concurrent mark)
  >  并发标记阶段就是进行GC Roots Tracing的过程
- 重新标记(CMS remark) 
  > 重新标记是为了修正并发标记期间用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段停顿的时间一般比初始标记阶段稍长一些，但远比并发标记的时间短
- 并发清除(CMS concurrent sweep)

其中初始标记、重新标记仍然需要"Stop The World"。

> 由于整个过程中耗时最长的并发标记和并发清除过程线程都可以与用户线程一起工作，所以，总体上说，CMS收集器的内存回收过程是与用户线程一起并发执行的

![image](https://user-images.githubusercontent.com/38010908/69699982-4ad21c00-1124-11ea-9894-f96ab5e3b240.png)

CMS的3个明显缺点。

1. CMS收集器对CPU资源非常敏感

在默认情况下，CMS默认启动的回收线程计算方式如下  
```
启动的回收线程数 = (CPU数量+3)/4
```
也就是在4核的情况下，会启动1条回收线程，在并发回收时回收线程会占用不少于25%的CPU资源，不过随着CPU的数量增加会下降。

但是，当CPU不足4个的时候，比如2个，那么则会占用不少于50%的运算能力去执行收集线程，这是无法忍受的。  
> 因此虚拟机提供了一种称为“增量式并发收集器”(I ncremental Concurrent Mark  Sweep/i-CMS)的CMS收集器变种，原理就是在并发回收的时候，用户线程跟回收线程交替执行，虽然回收的时间更长，但是对用户程序影响变小的。**但是实践证明，该增量CMS收集器效果很一般，以被声明`deprecated`，不再提倡用户使用了

2. CMS收集器无法处理浮动垃圾(Floating Garbage)，可能出现"Concurrent Mode Failure"失败而导致另一次Full GC的产生

**浮动垃圾**：  
由于CMS并发清理阶段用户线程还在运行着，伴随着程序运行自然还会有新的垃圾不断产生，这一部分垃圾在标记过程之后，CMS无法在当次收集中处理它们，只好留待下次GC时再清理掉。这部分垃圾就叫做浮动垃圾。


因为在垃圾收集阶段用户线程还需要运行，那也是需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间踢空并发收集时程序运行作使用。

这部分预留的空间可以通过`-XX:CMSInitiatingOccupancyFraction`的值进行修改。JDK1.6中，默认值为92%


当CMS运行期间预留的内存无法满足程序需要，就会出现一次"Concurrent Mode Failure"失败，这时虚拟机将启动后备预案：临时启用Serial Old收集器来重新进行老年代的垃圾收集，不过这样等待的时间就长了。

3. CMS是基于“标记-清除”算法实现的，意味着收集完之后，会有大量的空间碎片。

当碎片过多时，会出现尽管老年代还有很大的空间剩余，却无法找到足够大的连续空间来分配当前对象，不得不提前出发一次full GC。

为此，CMS提供了一个参数 `-XX:+UseCMSCompactAtFullCollection`用于开启CMS对空间进行整理,用于在CMS在顶不住要进行Full GC时开启内存碎片的合并整理过程，不过，整理过程是无法并发的，空间碎片问题没有了，不过停顿时间不得不变长了。


但是，虚拟机设计者又提供了另外一个参数`-XX:CMSFullGCsBeforeCompaction`这个参数用于设置执行多少次不压缩的Full GC后，跟着来一次带压缩的(默认为0，表示每次进入Full GC时都进行碎片整理)

## G1

G1是一款面向服务端应用的垃圾收集器，与其他GC收集器相比，G1具备如下特点：
- 并行与并发
  > G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短STW停顿的时间，部分其他收集器原本需要停顿java线程执行的GC动作，G1仍然可以通过并发的方式让java程序继续执行
- 分代收集
  > 虽然G1可以不需要其他收集器配合就能独立管理整个GC堆，但它能够采用不同的方式去处理新创建的对象和已经存活一段时间、熬过多次GC的旧对象以获取更好的收集效果
- 空间整合
  > G1整体来看是基于“标记-整理”算法实现的收集器，从局部(两个Region之间)上来看是基于"复制"算法实现的。但不管怎么说，G1在运作期间不会产生内存碎片空间，收集后能提供规整的可用内存
- 可预测的停顿
  > G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集器上的时间不得超过N毫秒。

在G1之前的其他收集器的范围都是整个新生代或者老年代，而G1不再是这样。在使用G1收集器时，java堆的内存布局就与其他收集器有很大差别，它将java堆划分为多个大小相等的独立区域(Region)。虽然还有新生代老年代的概念，但是不再是物理隔离了，他们都是一部分Region的集合。

G1建立可预测停顿时间模型的原因：

因为G1可以有计划地避免在整个java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的价值大小(回收所获得的空间大小以及回收所需时间的经验值)，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region。这种使用Region划分内存空间以及优先级的区域回收方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率。

> 在G1中，Region之间的对象引用以及其他收集器中的新生代与老年代之前的对象引用，虚拟机都是使用Remembered Set来避免扫描全堆的。  
> G1中每个Region都有一个与之对应的Remembered Set，虚拟机发现程序在对Reference类型的数据进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于不同的Region之中(在分代的例子中就是检查是否老年代中的对象引用了新生代中的对象)，如果是，便通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set中。当进行内存回收时，在GC根节点的枚举范围中加入Remembered Set即可保证不对全表扫描也不会有遗漏

[gc相关的、卡表说明](https://juejin.im/post/5b8d2a5551882542ba1ddcf8)


如果不计算维护Remembered Set的操作，G1收集器的运作大致可划分为一下几个步骤：
- 初始标记(Initial Marking)
- 并发标记(Concurrent Marking)
- 最终标记(Final Marking)
- 筛选回收(Live Data Counting and Evacuation)



![image](https://user-images.githubusercontent.com/38010908/69709345-2c761b80-1138-11ea-9c77-49e3005a9433.png)


