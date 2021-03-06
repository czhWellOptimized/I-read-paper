介绍了两个经典想法：多粒度锁和多种锁模式。

论文的前半部分

第一：如何执行互斥的锁定？什么时候使用粗粒度的锁，细粒度的锁？如何支持对于数据库不同层级的并发访问？

第二：介绍了多种隔离级别的概念。

论文的后半部分

提供了这些基于锁策略的基本做法。

还讨论了可恢复性的重要概念：可以在不影响其他事务的情况下中止本事务。

随着时代的发展，并发控制方法也在变化，但是有一点是确定的：没有一个单方面的最佳机制，每一个机制都是和负载量相关的。所以下一篇论文介绍了如何衡量并发控制的性能。

# Abstract

本文提出了一种将锁和资源集合联系起来的锁定协议。这个协议允许不同事务同时使用不同粒度的锁。同时该协议基于额外的锁模式以及传统的共享模式、互斥模式。这个协议是从对于锁的有向无环图、锁的动态图的简易层次种概括出来的。关于如何调度相同资源以及授予锁的问题之后会被讨论。最后，这些想法会和现存数据库提供的锁机制进行比较。

# 1. INTRODUCTION

第二节对可能的锁机制提供了简单的描述。

第三节概述了如何同时支持粗细可锁定对象组织进一个层次结构中。将这些想法概括成有向无环图。

第四节讨论了调度锁请求带来的问题。

第五节将这些想法和现存的数据库联系起来，并简单描述了我们在一个实验性数据库上是如何使用该锁协议的。

最后一节展示了该协议在层次结构上，网络中和关系系统中的应用。

# 2. GRANULARITY OF LOCKS

# 3. HIERARCHICAL LOCKS

通过比较谓词的方式进行锁定是比较麻烦的。

database、areas、files、records。既然是按层级加锁，那么如果对某一个节点加锁，对于它的子孙节点都要加锁。

我们的目标是<u>隐式</u>的锁定一棵子树。显示的锁定一棵子树就是锁定子树中的每一个节点，但是显示锁定开销太大了。

隐式的方式提出了一种叫做意向锁的东西。我们想要锁住一棵子树，就是要阻止祖先节点对于R或者R的子孙节点进行隐式的锁定。那么意向锁就是用意向锁锁定R的所有祖先节点，祖先节点被意向锁锁定说明此时低级别的层次中有锁存在，从而避免在祖先节点的隐式或者显示的锁定（我们认为锁定了祖先节点就是隐式的锁定了子孙节点）。

那么协议就是用I锁定R的所有祖先节点，并用S或X锁定R。

## 3.1 Access modes and compatibility

X,S,I互相都是不相容的，但是S和S之间是相容的，I和I之间也是相容的。

I锁又分为IS、IX。这么分的原因是IS可以与S相容，同时IS还可能转换成S。当然IX不能转换成S。

又介绍了SIX。假设一个事务想要读取一整个子树，并且修改子树中的特定节点。有以下几种方法：

1. 对root上X锁。
2. 对root上IX锁，并对root下的节点上I、S或者X。

第一种方法并发程度比较低，而如果只有一小部分的节点需要更新那么第二种方法开销就很大。

因此最好的方法是对子树上S和IX锁。S锁可以让事务读取，IX锁可以让事务对那些需要update的节点上X。

![image-20220308152736456](/Users/chenzhihao/Documents/chaosCode/I-read-paper/image/image-20220308152736456.png)

![image-20220308153813772](/Users/chenzhihao/Documents/chaosCode/I-read-paper/image/image-20220308153813772.png)

## 3.2 Rules for requesting nodes

1. 在请求R的S或者IS锁时，所有R的祖先节点必须持有IX或者IS
2. 在请求R的X、SIX、IX时，所有R的祖先节点都必须持有SIX、IX
3. 锁必须在事务结束的时候，或者从叶子到根的方式释放。如果锁不是在事务结束的时候释放，那么一定不能在释放祖先的锁时还持有着低级别的锁（意思是这时候要从叶子到根释放锁）。从叶子到根释放锁的原因：比如R现在上了X，R的祖先A上了IX。那么如果先释放IX，另外一个事务会认为此时A子树没有上X或者IX，然后事务直接上了X，此时R上了X，A也上了X，明显不对，因为A上的这个X蕴含的意思是也对A的子树上了X，然而一个节点是不可能同时上两个X的，所以出错！

锁请求是从根到叶子，释放是从叶子到根。

## 3.3 Several examples

## 3.4 Directed acyclic graphs of locks

## 3.5 The protocol for explicitly requesting locks on a DAG

1. 在对R上S或者IS之前，至少对R的一个parent上至少IS锁。
2. 在对R上IX，SIX或者X之前，对R的全部parent上至少IX锁。
3. 同3.2

## 3.6 Dynamic lock graphs

index interval locks 

4. 在lock graph种移动节点时，节点必须在图中过的旧位置和新位置上都隐式或者显式的上X锁。更进一步，不能移动成一个循环图

# 4. SCHEDULING AND GRANTING REQUESTS

FIFO。队列头已经被授予锁的叫做grant group。这个grant group有group mode，用来描述group可以兼容哪些lock。

当一个新请求到达队列时，有两种情况：有请求在等待了、所有请求都被授予。

当某个特殊的请求离开了group，group mode可能会改变，那么改变了以后就可能相容等待的请求。从论文中的例子可以看出，group mode的改变时尽可能的高级别，如果可以是IS或者S，那么应该变为S。

## 4.1 Conversion

![image-20220308192355975](/Users/chenzhihao/Documents/chaosCode/I-read-paper/image/image-20220308192355975.png)

<u>旧请求已经被授予</u>的情况下：

1. 如果新模式等于旧模式，那么立刻授予
2. 如果新模式和group其他请求像鹅绒，那么立刻授予，并且更新group mode（group mode级别会变高）
3. 其他情况下必须等待group中的其他请求和新模式相容

注意到这种立刻授予其实违背了公平调度的原则，因为理论上需要从group剔除旧请求，并将新请求追加到队列尾。

注意到死锁可能会出现。

## 4.2 Deadlock and lock thrashing

死锁，出现有向有环图，锁调度器绕着环找到回滚开销最小的替罪羊，让它回滚。

论文中提到的有关锁调度，检测，死锁恢复等问题，都属于低级别的调度决定。这些决定必须和高级别的调度器配合起来，否则容易出现长时间等待，锁抖动等现象，就像内存分配那样，既然内存已经怎么替换都不够用了，就暂时不给应用程序分配内存了，否则一直会出现一个页帧换来换去的现象。

# 5. LOCK HIERARCHIES IN EXISTING SYSTEMS

年代久远，就不看了。