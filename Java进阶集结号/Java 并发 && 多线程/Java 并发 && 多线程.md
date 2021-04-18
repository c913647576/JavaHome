### Java 并发 && 多线程
#### 1. synchronized 的实现原理以及锁优化？ ####

> 查看带有Synchronized语句块的class文件可以看到在同步代码块的起始位置插入了moniterenter指令，在同步代码块结束的位置插入了monitorexit指令。(JVM需要保证每一个monitorenter都有一个monitorexit与之相对应，但每个monitorexit不一定都有一个monitorenter)
但是查看同步方法的class文件时，同步方法并没有通过指令monitorenter和monitorexit来完成，而被翻译成普通的方法调用和返回指令，只是在其常量池中多了ACC_SYNCHRONIZED标示符。JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。
>     
>     synchronized的实现原理和应用总结
>     
>     （1）synchronized同步代码块：synchronized关键字经过编译之后，会在同步代码块前后分别形成
>     monitorenter和monitorexit字节码指令，在执行monitorenter指令的时候，首先尝试获取对象的锁，
>     如果这个锁没有被锁定或者当前线程已经拥有了那个对象的锁，锁的计数器就加1，在执行monitorexit指
>     令时会将锁的计数器减1，当减为0的时候就释放锁。如果获取对象锁一直失败，那当前线程就要阻塞等待，
>     直到对象锁被另一个线程释放为止。
>     （2）同步方法：方法级的同步是隐式的，无须通过字节码指令来控制，JVM可以从方法常量池的方法表结构
>     中的ACC_SYNCHRONIZED访问标志得知一个方法是否声明为同步方法。当方法调用的时，调用指令会检查方
>     法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先持有monitor对象，然后才
>     能执行方法，最后当方法执行完（无论是正常完成还是非正常完成）时释放monitor对象。在方法执行期间，
>     执行线程持有了管程，其他线程都无法再次获取同一个管程。

> moniterenter和moniterexit指令是通过monitor对象实现的。

> Synchronized的实现不仅与monitor对象有关，还与另一个东西密切相关，那就是对象头。

> 每个对象都有一个监视器锁(monitor)与之对应。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：
> 
> 1、如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
> 
> 2、如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
> 
> 3.如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

> Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。

> **Java对象头**

> 每个对象分为三块区域:对象头、实例数据和对齐填充。

>     对象头包含两部分，第一部分是Mark Word，用于存储对象自身的运行时数据，如哈希码（HashCode）、
>     GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等，这一部分占一个字节。第二部
>     分是Klass Pointer（类型指针），是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象
>     是哪个类的实例，这部分也占一个字节。(如果对象是数组类型的，则需要3个字节来存储对象头，因为还需
>     要一个字节存储数组的长度)
>     
>     实例数据存放的是类属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部
>     分内存按4字节对齐。
>     
>     填充数据是因为虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节
>     对齐。

> 对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。例如在32位的HotSpot虚拟机 中对象未被锁定的状态下，Mark Word的32个Bits空间中的25Bits用于存储对象哈希码(HashCode)，4Bits用于存储对象分代年龄，2Bits用于存储锁标志 位，1Bit固定为0，在其他状态(轻量级锁定、重量级锁定、GC标记、可偏向)下对象的存储内容如下表所示。

> ![](https://img-blog.csdnimg.cn/2019061215553348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppbmppbmlhbzE=,size_16,color_FFFFFF,t_70)

> 从对象头的存储内容可以看出锁的状态都保存在对象头中，Synchronized也不例外，当其从轻量级锁膨胀为重量级锁时，锁标识位为10，其中指针指向的是monitor对象(也称为管程或监视器锁)的起始地址。
> 
> 关于Synchronized的实现在java对象头里较为简单，只是改变一下标识位，并将指针指向monitor对象的起始地址，其实现的重点是monitor对象。

> **Monitor对象**
> 
> 什么是Monitor？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。
> 在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下(位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的)
> ObjectMonitor中有几个关键属性，

> - _count用来记录该线程获取锁的次数
> - _WaitSet存放处于wait状态的线程队列
> - _EntryList存放处于等待获取锁block状态的线程队列，即被阻塞的线程
> - _owner指向持有ObjectMonitor对象的线程

> 当多个线程同时访问一段同步代码时，首先会进入_EntryList队列中，当某个线程获取到对象的monitor后进入_Owner区域并把monitor中的_owner变量设置为当前线程，同时monitor中的计数器_count加1，若线程调用wait()方法，将释放当前持有的monitor，_owner变量恢复为null，_count自减1，同时该线程进入_WaitSet集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示

> ![](https://img-blog.csdnimg.cn/2019061215565082.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppbmppbmlhbzE=,size_16,color_FFFFFF,t_70)

> **Synchronized优化**

> 早期，Synchronized属于重量级锁，效率低下，因为监视器锁(monitor)是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。庆幸的是在Java 6之后Java官方对从JVM层面对synchronized较大优化，所以现在的synchronized锁效率也优化得很不错了，Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了偏向锁、轻量级锁和自旋锁等概念，接下来我们将简单了解一下Java官方在JVM层面对Synchronized锁的优化。

> 
1. 偏向锁
> 
> 引入偏向锁的主要原因是，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。
> 
> 引入的主要目的是，为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径。因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。
> 
> 偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提升程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。
> 
> 但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。

> 其执行流程为：
> 
> 获取锁
> 
>     检测Mark Word是否为可偏向状态，即是否为偏向锁1，锁标识位为01；
>     
>     若为可偏向状态，则测试线程ID是否为当前线程ID，如果是，则执行步骤(5)，否则执行步骤(3)；
>     
>     如果线程ID不为当前线程ID，则通过CAS操作竞争锁，竞争成功，则将Mark Word的线程ID替换为当前线
>     程ID，否则执行线程(4)；
>     
>     通过CAS竞争锁失败，证明当前存在多线程竞争情况，当到达全局安全点，获得偏向锁的线程被挂起，偏向
>     锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码块；
>     
>     执行同步代码块
> 
> 释放锁
> 
> 偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点（这个时间点是上没有正在执行的代码）。其步骤如下：
>     
>     暂停拥有偏向锁的线程，判断锁对象石是否还处于被锁定状态；
>     
>     撤销偏向苏，恢复到无锁状态(01)或者轻量级锁的状态；

> 那么轻量级锁和偏向锁的使用场景：
> 轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能。

> 2. 轻量级锁

> 引入轻量级锁的主要原因是，对绝大部分的锁，在整个同步周期内都不存在竞争，可能是交替获取锁然后执行。(与偏向锁的区别是，引入偏向锁是假设同一个锁都是由同一线程多次获得，而轻量级锁是假设同一个锁是由n个线程交替获得；相同点是都是假设不存在多线程竞争)
> 
> 引入轻量级锁的主要目的是，在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗(多指时间消耗)。
> 
> 触发轻量级锁的条件是当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁，此时Mark Word的结构也变为轻量级锁的结构。如果存在多个线程同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

> 其步骤如下：
> 
> 获取锁

>     判断当前对象是否处于无锁状态（hashcode、0、01），若是，则JVM首先将在当前线程的栈帧中建立一
>     个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了
>     一个Displaced前缀，即Displaced Mark Word）；否则执行步骤（3）；
>     
>     JVM利用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，如果成功表示竞争到锁，则
>     将锁标志位变成00（表示此对象处于轻量级锁状态），执行同步操作；如果失败则执行步骤（3）；
>     
>     判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则
>     直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁，
>     锁标志位变成10，后面等待的线程将会进入阻塞状态；

> 释放锁
> 
> 轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下：


>     取出在获取轻量级锁保存在Displaced Mark Word中的数据；
>     
>     用CAS操作将取出的数据替换当前对象的Mark Word中，如果成功，则说明释放锁成功，否则执行（3）；
>     
>     如果CAS操作替换失败，说明有其他线程尝试获取该锁，则需要在释放锁的同时需要唤醒被挂起的线程。


> 3. 自旋锁

> 线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时我们发现在许多应用上面，对象锁的锁状态只会持续很短一段时间，为了这一段很短的时间频繁地阻塞和唤醒线程是非常不值得的。所以引入自旋锁。
> 
> 何谓自旋锁？
> 
> 所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。怎么等待呢？执行一段无意义的循环即可（自旋）。
> 
> 自旋等待不能替代阻塞，虽然它可以避免线程切换带来的开销，但是它占用了处理器的时间。如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好，反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作，这样反而会带来性能上的浪费。所以说，自旋等待的时间（自旋的次数）必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。
> 
> 自旋锁在JDK 1.4.2中引入，默认关闭，但是可以使用-XX:+UseSpinning开开启，在JDK1.6中默认开启。同时自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整；
> 
> 如果通过参数-XX:preBlockSpin来调整自旋锁的自旋次数，会带来诸多不便。假如我将参数调整为10，但是很多线程都是等你刚刚退出自旋的时候就释放了锁（假如你再多自旋一两次就可以获取锁），你是不是很尴尬。于是JDK1.6引入自适应的自旋锁，让虚拟机会变得越来越聪明。
> 
> JDK 1.6引入了更加聪明的自旋锁，即自适应自旋锁。所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。它怎么做呢？线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。
> 
> 轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。如果自旋之后依然没有获取到锁，也就只能升级为重量级锁了。

> 整个执行流程如下：

>     当一个线程(假设叫A线程)想要获得锁时，首先检查对象头中的锁标志，如果是偏向锁，则跳转到2,如果是
>     无锁状态，则跳转到
>     
>     检查对象头中的偏向线程id是否指向A线程，是,则直接执行同步代码块,不是则3。
>     
>     使用cas操作将替换对象头中的偏向线程id,成功，则直接执行同步代码块。失败则说明其他的线程(假设
>     叫B线程)已经拥有偏向锁了,那么进行偏向锁的撤销(因为这里有竞争了)，此时执行4。
>     
>     B线程运行到全局安全点后，暂停该线程，检查它的状态,如果处于不活动或者已经退出同步代码块则原持
>     有偏向锁的线程释放锁，然后A再次执行3。如果仍处于活动状态，则需要升级为轻量级锁，此时执行5。
>     
>     在B线程的栈中分配锁记录，拷贝对象头中的MarkWord到锁记录中，然后将MarkWord改为指向B线程，同
>     时将对象头中的锁标志信息改为轻量级锁的00,然后唤醒B线程，也就是从安全点处继续执行。
>     
>     由于锁升级为轻量级锁, A线程也进行相同的操作，即，在A线程的栈中分配锁记录，拷贝对象头中的
>     Mark Word到锁记录中，然后使用cas操作替换MarkWord,因为此时B线程拥有锁，因此, A线程自旋。如
>     果自旋一定次数内成功获得锁，那么A线程获得轻量级锁，执行同步代码块。若自旋后仍未获得锁，A升级
>     为重量级锁，将对象头中的锁标志信息改为重量级的10，同时阻塞,此时请看7。
>     
>     B线程在释放锁的时候，使用cas将MarkWord中的信息替换，成功，则表示无竞争(这个时候还是轻量级锁,
>      A线程可能正在自旋中)直接释放。失败(因为这个时候锁已经膨胀)，那么释放之 后唤醒被挂起的线程(在
>      这个例子中，也就是A)。

> 4. 其他优化:
> 
>     自旋与自适应自旋：
>     
>     如果持有锁的线程能在很短时间内释放锁资源，就可以让线程执行一个忙循环（自旋），等持有锁的线程释放锁后即可立即获取锁，这样就避免用户线程和内核的切换的消耗。但是线程自旋需要消耗cpu的资源，如果一直得不到锁就会浪费cpu资源。因此在jdk1.6引入了自适应自旋锁，自旋等待的时候不固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。
>     
>     锁消除
>     
>     锁消除是指虚拟机即时编译器在运行时，对于一些代码上要求同步但是被检测不可能存在共享数据竞争的锁进行消除。例如String类型的连接操作，String是一个不可变对象，字符串的连接操作总是通过生成新的String对象来进行的，Javac编译器会对String连接做自动优化，在JDK1.5的版本中使用的是StringBuffer对象的append操作，StringBuffer的append方法是同步方法，这段代码在经过即时编译器编译之后就会忽略掉所有的同步直接执行。在JDK1.5之后是使用的StringBuilder对象的append操作来优化字符串连接的。
>     
>     锁粗化
>     
>     将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁。例如每次调用StringBuffer.append方法都需要加锁，如果虚拟机检测到有一系列的连续操作都是对同一个对象反复加锁和解锁，就会将其合并成一个更大范围的加锁和解锁操作。

#### 2. ThreadLocal原理，使用注意点，应用场景有哪些？ ####

</span><span class="suffix" style="display: none;"></span></h3>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">回答四个主要点：</p>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ThreadLocal是什么?</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ThreadLocal原理</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ThreadLocal使用注意点</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ThreadLocal的应用场景</section></li></ul>
<h4 data-tool="mdnice编辑器" style="margin-top: 30px; margin-bottom: 15px; padding: 0px; font-weight: bold; color: black; font-size: 18px;"><span class="prefix" style="display: none;"></span><span class="content" style="height: 16px; line-height: 16px; font-size: 16px;"><span style="background-image: url(https://imgkr.cn-bj.ufileos.com/899e43b7-5a08-4ac6-aa00-1c45f169a65b.png); display: inline-block; width: 16px; height: 16px; background-size: 100%; background-position: left bottom; background-repeat: no-repeat; width: 16px; height: 15px; line-height: 15px; margin-right: 6px; margin-bottom: -2px;"></span>ThreadLocal是什么?</span><span class="suffix" style="display: none;"></span></h4>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">ThreadLocal，即线程本地变量。如果你创建了一个ThreadLocal变量，那么访问这个变量的每个线程都会有这个变量的一个本地拷贝，多个线程操作这个变量的时候，实际是操作自己本地内存里面的变量，从而起到线程隔离的作用，避免了线程安全问题。</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">//创建一个ThreadLocal变量
<span/>static ThreadLocal&lt;String&gt; localVariable = new ThreadLocal&lt;&gt;();
<span/></code></pre>
<h4 data-tool="mdnice编辑器" style="margin-top: 30px; margin-bottom: 15px; padding: 0px; font-weight: bold; color: black; font-size: 18px;"><span class="prefix" style="display: none;"></span><span class="content" style="height: 16px; line-height: 16px; font-size: 16px;"><span style="background-image: url(https://imgkr.cn-bj.ufileos.com/899e43b7-5a08-4ac6-aa00-1c45f169a65b.png); display: inline-block; width: 16px; height: 16px; background-size: 100%; background-position: left bottom; background-repeat: no-repeat; width: 16px; height: 15px; line-height: 15px; margin-right: 6px; margin-bottom: -2px;"></span>ThreadLocal原理</span><span class="suffix" style="display: none;"></span></h4>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">ThreadLocal内存结构图：</p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><img src="https://user-gold-cdn.xitu.io/2020/7/26/1738a130b3c6e020?w=1211&amp;h=982&amp;f=png&amp;s=257051" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;">
由结构图是可以看出：</p>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">Thread对象中持有一个ThreadLocal.ThreadLocalMap的成员变量。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ThreadLocalMap内部维护了Entry数组，每个Entry代表一个完整的对象，key是ThreadLocal本身，value是ThreadLocal的泛型值。</section></li></ul>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">对照着几段关键源码来看，更容易理解一点哈~</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">public class Thread implements Runnable {
<span/>   //ThreadLocal.ThreadLocalMap是Thread的属性
<span/>   ThreadLocal.ThreadLocalMap threadLocals = null;
<span/>}
<span/></code></pre>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">ThreadLocal中的关键方法set()和get()</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">    public void <span class="hljs-built_in" style="color: #0086b3; line-height: 26px;">set</span>(T value) {
<span/>        Thread t = Thread.currentThread(); //获取当前线程t
<span/>        ThreadLocalMap map = getMap(t);  //根据当前线程获取到ThreadLocalMap
<span/>        <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">if</span> (map != null)
<span/>            map.set(this, value); //K，V设置到ThreadLocalMap中
<span/>        <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">else</span>
<span/>            createMap(t, value); //创建一个新的ThreadLocalMap
<span/>    }
<span/>
<span/>    public T <span class="hljs-function" style="line-height: 26px;"><span class="hljs-title" style="color: #900; font-weight: bold; line-height: 26px;">get</span></span>() {
<span/>        Thread t = Thread.currentThread();//获取当前线程t
<span/>        ThreadLocalMap map = getMap(t);//根据当前线程获取到ThreadLocalMap
<span/>        <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">if</span> (map != null) {
<span/>            //由this（即ThreadLoca对象）得到对应的Value，即ThreadLocal的泛型值
<span/>            ThreadLocalMap.Entry e = map.getEntry(this);
<span/>            <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">if</span> (e != null) {
<span/>                @SuppressWarnings(<span class="hljs-string" style="color: #d14; line-height: 26px;">"unchecked"</span>)
<span/>                T result = (T)e.value; 
<span/>                <span class="hljs-built_in" style="color: #0086b3; line-height: 26px;">return</span> result;
<span/>            }
<span/>        }
<span/>        <span class="hljs-built_in" style="color: #0086b3; line-height: 26px;">return</span> setInitialValue();
<span/>    }
<span/></code></pre>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">ThreadLocalMap的Entry数组</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">static class ThreadLocalMap {
<span/>    static class Entry extends WeakReference&lt;ThreadLocal&lt;?&gt;&gt; {
<span/>        /** The value associated with this ThreadLocal. */
<span/>        Object value;
<span/>
<span/>        Entry(ThreadLocal&lt;?&gt; k, Object v) {
<span/>            super(k);
<span/>            value = v;
<span/>        }
<span/>    }
<span/>}
<span/></code></pre>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">所以怎么回答<strong style="color: #3594F7; font-weight: bold;"><span>「</span>ThreadLocal的实现原理<span>」</span></strong>？如下，最好是能结合以上结构图一起说明哈~</p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<ul style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，即每个线程都有一个属于自己的ThreadLocalMap。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ThreadLocalMap内部维护着Entry数组，每个Entry代表一个完整的对象，key是ThreadLocal本身，value是ThreadLocal的泛型值。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">每个线程在往ThreadLocal里设置值的时候，都是往自己的ThreadLocalMap里存，读也是以某个ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。</section></li></ul>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<h4 data-tool="mdnice编辑器" style="margin-top: 30px; margin-bottom: 15px; padding: 0px; font-weight: bold; color: black; font-size: 18px;"><span class="prefix" style="display: none;"></span><span class="content" style="height: 16px; line-height: 16px; font-size: 16px;"><span style="background-image: url(https://imgkr.cn-bj.ufileos.com/899e43b7-5a08-4ac6-aa00-1c45f169a65b.png); display: inline-block; width: 16px; height: 16px; background-size: 100%; background-position: left bottom; background-repeat: no-repeat; width: 16px; height: 15px; line-height: 15px; margin-right: 6px; margin-bottom: -2px;"></span>ThreadLocal 内存泄露问题</span><span class="suffix" style="display: none;"></span></h4>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">先看看一下的TreadLocal的引用示意图哈，</p>
<figure data-tool="mdnice编辑器" style="margin: 0; margin-top: 10px; margin-bottom: 10px; display: flex; flex-direction: column; justify-content: center; align-items: center;"><img src="https://user-gold-cdn.xitu.io/2020/7/26/1738b3cf19130e19?w=1804&amp;h=741&amp;f=png&amp;s=99228" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></figure>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">ThreadLocalMap中使用的 key 为 ThreadLocal 的弱引用，如下
<img src="https://user-gold-cdn.xitu.io/2020/7/26/1738b1a2f8978e47?w=1052&amp;h=498&amp;f=png&amp;s=48612" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<p style="padding-top: 8px; padding-bottom: 8px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px; margin: 0px; line-height: 26px; color: #595959;">弱引用：只要垃圾回收机制一运行，不管JVM的内存空间是否充足，都会回收该对象占用的内存。</p>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">弱引用比较容易被回收。因此，如果ThreadLocal（ThreadLocalMap的Key）被垃圾回收器回收了，但是因为ThreadLocalMap生命周期和Thread是一样的，它这时候如果不被回收，就会出现这种情况：ThreadLocalMap的key没了，value还在，这就会<strong style="color: #3594F7; font-weight: bold;"><span>「</span>造成了内存泄漏问题<span>」</span></strong>。</p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">如何<strong style="color: #3594F7; font-weight: bold;"><span>「</span>解决内存泄漏问题<span>」</span></strong>？使用完ThreadLocal后，及时调用remove()方法释放内存空间。</p>
<h4 data-tool="mdnice编辑器" style="margin-top: 30px; margin-bottom: 15px; padding: 0px; font-weight: bold; color: black; font-size: 18px;"><span class="prefix" style="display: none;"></span><span class="content" style="height: 16px; line-height: 16px; font-size: 16px;"><span style="background-image: url(https://imgkr.cn-bj.ufileos.com/899e43b7-5a08-4ac6-aa00-1c45f169a65b.png); display: inline-block; width: 16px; height: 16px; background-size: 100%; background-position: left bottom; background-repeat: no-repeat; width: 16px; height: 15px; line-height: 15px; margin-right: 6px; margin-bottom: -2px;"></span>ThreadLocal的应用场景</span><span class="suffix" style="display: none;"></span></h4>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">数据库连接池</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">会话管理中使用</section></li></ul>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

#### 3. synchronized和ReentrantLock的区别？ ####

</span><span class="suffix" style="display: none;"></span></h3>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">我记得校招的时候，这道面试题出现的频率还是挺高的~可以从锁的实现、功能特点、性能等几个维度去回答这个问题，</p>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>锁的实现：<span>」</span></strong> synchronized是Java语言的关键字，基于JVM实现。而ReentrantLock是基于JDK的API层面实现的（一般是lock()和unlock()方法配合try/finally 语句块来完成。）</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>性能：<span>」</span></strong> 在JDK1.6锁优化以前，synchronized的性能比ReenTrantLock差很多。但是JDK6开始，增加了适应性自旋、锁消除等，两者性能就差不多了。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>功能特点：<span>」</span></strong> ReentrantLock 比 synchronized 增加了一些高级功能，如等待可中断、可实现公平锁、可实现选择性通知。</section></li></ul>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<ul style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ReentrantLock提供了一种能够中断等待锁的线程的机制，通过lock.lockInterruptibly()来实现这个机制。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ReentrantLock可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">synchronized与wait()和notify()/notifyAll()方法结合实现等待/通知机制，ReentrantLock类借助Condition接口与newCondition()方法实现。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">ReentrantLock需要手工声明来加锁和释放锁，一般跟finally配合释放锁。而synchronized不用手动释放锁。</section></li></ul>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

#### 4. 说说CountDownLatch与CyclicBarrier 区别 ####

</span><span class="suffix" style="display: none;"></span></h3>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">CountDownLatch：一个或者多个线程，等待其他多个线程完成某件事情之后才能执行;</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">CyclicBarrier：多个线程互相等待，直到到达同一个同步点，再继续一起执行。
<img src="https://user-gold-cdn.xitu.io/2020/7/27/1738d9ab805c995c?w=1165&amp;h=785&amp;f=png&amp;s=104903" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></section></li></ul>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">举个例子吧：</p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<ul style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">CountDownLatch：假设老师跟同学约定周末在公园门口集合，等人齐了再发门票。那么，发门票（这个主线程），需要等各位同学都到齐（多个其他线程都完成），才能执行。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">CyclicBarrier:多名短跑运动员要开始田径比赛，只有等所有运动员准备好，裁判才会鸣枪开始，这时候所有的运动员才会疾步如飞。</section></li></ul>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

#### 5. Fork/Join框架的理解 ####

</span><span class="suffix" style="display: none;"></span></h3>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<p style="padding-top: 8px; padding-bottom: 8px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px; margin: 0px; line-height: 26px; color: #595959;">Fork/Join框架是Java7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。</p>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">Fork/Join框架需要理解两个点，<strong style="color: #3594F7; font-weight: bold;"><span>「</span>分而治之<span>」</span></strong>和<strong style="color: #3594F7; font-weight: bold;"><span>「</span>工作窃取算法<span>」</span></strong>。</p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>分而治之<span>」</span></strong></p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">以上Fork/Join框架的定义，就是分而治之思想的体现啦
<img src="https://user-gold-cdn.xitu.io/2020/7/27/17390bea64fadb1f?w=1295&amp;h=1006&amp;f=png&amp;s=69986" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>工作窃取算法<span>」</span></strong></p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">把大任务拆分成小任务，放到不同队列执行，交由不同的线程分别执行时。有的线程优先把自己负责的任务执行完了，其他线程还在慢慢悠悠处理自己的任务，这时候为了充分提高效率，就需要工作盗窃算法啦~</p>
<figure data-tool="mdnice编辑器" style="margin: 0; margin-top: 10px; margin-bottom: 10px; display: flex; flex-direction: column; justify-content: center; align-items: center;"><img src="https://user-gold-cdn.xitu.io/2020/7/27/17390d4b199b668a?w=813&amp;h=576&amp;f=png&amp;s=65266" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></figure>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">工作盗窃算法就是，<strong style="color: #3594F7; font-weight: bold;"><span>「</span>某个线程从其他队列中窃取任务进行执行的过程<span>」</span></strong>。一般就是指做得快的线程（盗窃线程）抢慢的线程的任务来做，同时为了减少锁竞争，通常使用双端队列，即快线程和慢线程各在一端。</p>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

#### 6.  为什么我们调用start()方法时会执行run()方法，为什么我们不能直接调用run()方法？ ####

</span><span class="suffix" style="display: none;"></span></h3>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">看看Thread的start方法说明哈~</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">    /**
<span/>     * Causes this thread to begin execution; the Java Virtual Machine
<span/>     * calls the &lt;code&gt;run&lt;/code&gt; method of this thread.
<span/>     * &lt;p&gt;
<span/>     * The result is that two threads are running concurrently: the
<span/>     * current thread (<span class="hljs-built_in" style="color: #0086b3; line-height: 26px;">which</span> returns from the call to the
<span/>     * &lt;code&gt;start&lt;/code&gt; method) and the other thread (<span class="hljs-built_in" style="color: #0086b3; line-height: 26px;">which</span> executes its
<span/>     * &lt;code&gt;run&lt;/code&gt; method).
<span/>     * &lt;p&gt;
<span/>     * It is never legal to start a thread more than once.
<span/>     * In particular, a thread may not be restarted once it has completed
<span/>     * execution.
<span/>     *
<span/>     * @exception  IllegalThreadStateException  <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">if</span> the thread was already
<span/>     *               started.
<span/>     * @see        <span class="hljs-comment" style="color: #998; font-style: italic; line-height: 26px;">#run()</span>
<span/>     * @see        <span class="hljs-comment" style="color: #998; font-style: italic; line-height: 26px;">#stop()</span>
<span/>     */
<span/>    public synchronized void <span class="hljs-function" style="line-height: 26px;"><span class="hljs-title" style="color: #900; font-weight: bold; line-height: 26px;">start</span></span>() {
<span/>     ......
<span/>    }
<span/></code></pre>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">JVM执行start方法，会另起一条线程执行thread的run方法，这才起到多线程的效果~ <strong style="color: #3594F7; font-weight: bold;"><span>「</span>为什么我们不能直接调用run()方法？<span>」</span></strong>
如果直接调用Thread的run()方法，其方法还是运行在主线程中，没有起到多线程效果。</p>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

#### 7. Java中的volatile关键是什么作用？怎样使用它？在Java中它跟synchronized方法有什么不同？volatile 的实现原理 ####

> voliate关键字的两个作用：1、 保证变量的内存可见性；2、 禁止指令重排序；
> 
> volatile只能保证可见性和有序性（先行发生），也可以保证单次读/写的原子性，如 i = 1、isSuccess = false等操作，但是并不能保证 i++这种操作的原子性，因为本质上 i++是读、写两次操作。synchronized可以保证原子性、可见性及有序性。volatile在某些场景下可以代替 Synchronized。但是,volatile 的不能完全取代 Synchronized 的位置，只有在一些特殊的场景下，才能适用 volatile。总的来说，必须同时满足下面两个条件才能保证在并发环境的线程安
全：（1）对变量的写操作不依赖于当前值（比如 i++），或者说是单纯的变量赋值（boolean
flag = true） 。（2）该变量没有包含在具有其他变量的不变式中， 也就是说，不同的 volatile 变量之间，不能互相依赖。 只有在状态真正独立于程序内其他内容时才能使用 volatile。
> 
> 实现原理：主内存与工作内存的工作原理。
>
#### 8. CAS？CAS 有什么缺陷，如何解决？ ####

</span><span class="suffix" style="display: none;"></span></h3>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">CAS,Compare and Swap，比较并交换；</p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<p style="padding-top: 8px; padding-bottom: 8px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px; margin: 0px; line-height: 26px; color: #595959;">CAS 涉及3个操作数，内存地址值V，预期原值A，新值B；
如果内存位置的值V与预期原A值相匹配，就更新为新值B，否则不更新</p>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">CAS有什么缺陷？</p>
<figure data-tool="mdnice编辑器" style="margin: 0; margin-top: 10px; margin-bottom: 10px; display: flex; flex-direction: column; justify-content: center; align-items: center;"><img src="https://user-gold-cdn.xitu.io/2020/7/28/17392a38b3e75683?w=804&amp;h=431&amp;f=png&amp;s=46436" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></figure>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>ABA 问题<span>」</span></strong></p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<p style="padding-top: 8px; padding-bottom: 8px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px; margin: 0px; line-height: 26px; color: #595959;">并发环境下，假设初始条件是A，去修改数据时，发现是A就会执行修改。但是看到的虽然是A，中间可能发生了A变B，B又变回A的情况。此时A已经非彼A，数据即使成功修改，也可能有问题。</p>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">可以通过AtomicStampedReference<strong style="color: #3594F7; font-weight: bold;"><span>「</span>解决ABA问题<span>」</span></strong>，它，一个带有标记的原子引用类，通过控制变量值的版本来保证CAS的正确性。</p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>循环时间长开销<span>」</span></strong></p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<p style="padding-top: 8px; padding-bottom: 8px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px; margin: 0px; line-height: 26px; color: #595959;">自旋CAS，如果一直循环执行，一直不成功，会给CPU带来非常大的执行开销。</p>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">很多时候，CAS思想体现，是有个自旋次数的，就是为了避开这个耗时问题~</p>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>只能保证一个变量的原子操作。<span>」</span></strong></p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<p style="padding-top: 8px; padding-bottom: 8px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px; margin: 0px; line-height: 26px; color: #595959;">CAS 保证的是对一个变量执行操作的原子性，如果对多个变量操作时，CAS 目前无法直接保证操作的原子性的。</p>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">可以通过这两个方式解决这个问题：</p>
<blockquote data-tool="mdnice编辑器" style="display: block; font-size: 0.9em; overflow: auto; overflow-scrolling: touch; padding-top: 10px; padding-bottom: 10px; padding-left: 20px; padding-right: 10px; margin-bottom: 20px; margin-top: 20px; text-size-adjust: 100%; line-height: 1.55em; font-weight: 400; border-radius: 6px; color: #595959; font-style: normal; text-align: left; box-sizing: inherit; border-left: none; border: 1px solid RGBA(64, 184, 250, .4); background: RGBA(64, 184, 250, .1);"><span style="color: RGBA(64, 184, 250, .5); font-size: 34px; line-height: 1; font-weight: 700;">❝</span>
<ul style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">使用互斥锁来保证原子性；</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">将多个变量封装成对象，通过AtomicReference来保证原子性。</section></li></ul>
<span style="float: right; color: RGBA(64, 184, 250, .5);">❞</span></blockquote>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">有兴趣的朋友可以看看我之前的这篇实战文章哈~
<span class="footnote-word" style="font-weight: normal; color: #595959;">CAS乐观锁解决并发问题的一次实践</span><sup class="footnote-ref" style="line-height: 0; font-weight: normal; color: #595959;">[2]</sup></p>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

#### 9. 如何检测死锁？怎么预防死锁？死锁四个必要条件 ####

</span><span class="suffix" style="display: none;"></span></h3>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">死锁是指多个线程因竞争资源而造成的一种互相等待的僵局。如图感受一下：
<img src="https://user-gold-cdn.xitu.io/2020/7/28/17392c9774168d4c?w=1250&amp;h=830&amp;f=png&amp;s=186643" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;">
<strong style="color: #3594F7; font-weight: bold;"><span>「</span>死锁的四个必要条件：<span>」</span></strong></p>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">互斥：一次只有一个进程可以使用一个资源。其他进程不能访问已分配给其他进程的资源。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">占有且等待：当一个进程在等待分配得到其他资源时，其继续占有已分配得到的资源。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">非抢占：不能强行抢占进程中已占有的资源。</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">循环等待：存在一个封闭的进程链，使得每个资源至少占有此链中下一个进程所需要的一个资源。</section></li></ul>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;"><strong style="color: #3594F7; font-weight: bold;"><span>「</span>如何预防死锁？<span>」</span></strong></p>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">加锁顺序（线程按顺序办事）</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">加锁时限 （线程请求所加上权限，超时就放弃，同时释放自己占有的锁）</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">死锁检测</section></li></ul>

#### 10. 如果线程过多,会怎样? ####
#### 11. 说说 Semaphore原理？ ####
#### 12. AQS组件，实现原理 ####
#### 13. 假设有T1、T2、T3三个线程，你怎样保证T2在T1执行完后执行，T3在T2执行完后执行？ ####
#### 14. LockSupport作用是？ ####
#### 15. Condition接口及其实现原理 ####
#### 16. 说说并发与并行的区别? ####
#### 17. 为什么要用线程池？Java的线程池内部机制，参数作用，几种工作阻塞队列，线程池类型以及使用场景 ####
#### 18. 如何保证多线程下 i++ 结果正确？ ####

</span><span class="suffix" style="display: none;"></span></h3>
<figure data-tool="mdnice编辑器" style="margin: 0; margin-top: 10px; margin-bottom: 10px; display: flex; flex-direction: column; justify-content: center; align-items: center;"><img src="https://user-gold-cdn.xitu.io/2020/7/28/17392b5e8436976e?w=935&amp;h=464&amp;f=png&amp;s=55515" alt style="max-width: 100%; border-radius: 6px; display: block; margin: 20px auto; object-fit: contain; box-shadow: 2px 4px 7px #999;"></figure>
<ul data-tool="mdnice编辑器" style="margin-top: 8px; margin-bottom: 8px; padding-left: 25px; font-size: 15px; color: #595959; list-style-type: circle;">
<li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">使用循环CAS，实现i++原子操作</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">使用锁机制，实现i++原子操作</section></li><li><section style="margin-top: 5px; margin-bottom: 5px; line-height: 26px; text-align: left; font-size: 14px; font-weight: normal; color: #595959;">使用synchronized，实现i++原子操作</section></li></ul>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">没有代码demo，感觉是没有灵魂的~ 如下：</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">/**
<span/> *  @Author 捡田螺的小男孩
<span/> */
<span/>public class AtomicIntegerTest {
<span/>
<span/>    private static AtomicInteger atomicInteger = new AtomicInteger(0);
<span/>
<span/>    public static void main(String[] args) throws InterruptedException {
<span/>        testIAdd();
<span/>    }
<span/>
<span/>    private static void testIAdd() throws InterruptedException {
<span/>        //创建线程池
<span/>        ExecutorService executorService = Executors.newFixedThreadPool(2);
<span/>        <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">for</span> (int i = 0; i &lt; 1000; i++) {
<span/>            executorService.execute(() -&gt; {
<span/>                <span class="hljs-keyword" style="color: #333; font-weight: bold; line-height: 26px;">for</span> (int j = 0; j &lt; 2; j++) {
<span/>                    //自增并返回当前值
<span/>                    int andIncrement = atomicInteger.incrementAndGet();
<span/>                    System.out.println(<span class="hljs-string" style="color: #d14; line-height: 26px;">"线程:"</span> + Thread.currentThread().getName() + <span class="hljs-string" style="color: #d14; line-height: 26px;">" count="</span> + andIncrement);
<span/>                }
<span/>            });
<span/>        }
<span/>        executorService.shutdown();
<span/>        Thread.sleep(100);
<span/>        System.out.println(<span class="hljs-string" style="color: #d14; line-height: 26px;">"最终结果是 ："</span> + atomicInteger.get());
<span/>    }
<span/>    
<span/>}
<span/></code></pre>
<p data-tool="mdnice编辑器" style="padding-top: 8px; padding-bottom: 8px; line-height: 26px; color: #2b2b2b; margin: 10px 0px; letter-spacing: 2px; font-size: 14px; word-spacing: 2px;">运行结果：</p>
<pre class="custom" data-tool="mdnice编辑器" style="margin-top: 10px; margin-bottom: 10px; border-radius: 5px; box-shadow: rgba(0, 0, 0, 0.55) 0px 2px 10px;"><span style="display: block; background: url(https://imgkr.cn-bj.ufileos.com/97e4eed2-a992-4976-acf0-ccb6fb34d308.png); height: 30px; width: 100%; background-size: 40px; background-repeat: no-repeat; background-color: #f8f8f8; margin-bottom: -7px; border-radius: 5px; background-position: 10px 10px;"></span><code class="hljs" style="overflow-x: auto; padding: 16px; color: #333; display: block; font-family: Operator Mono, Consolas, Monaco, Menlo, monospace; font-size: 12px; -webkit-overflow-scrolling: touch; letter-spacing: 0px; padding-top: 15px; background: #f8f8f8; border-radius: 5px;">...
<span/>线程:pool-1-thread-1 count=1997
<span/>线程:pool-1-thread-1 count=1998
<span/>线程:pool-1-thread-1 count=1999
<span/>线程:pool-1-thread-2 count=315
<span/>线程:pool-1-thread-2 count=2000
<span/>最终结果是 ：2000
<span/></code></pre>
<h3 data-tool="mdnice编辑器" style="padding: 0px; color: black; font-size: 17px; font-weight: bold; text-align: center; position: relative; margin-top: 20px; margin-bottom: 20px;"><span class="prefix" style="display: none;"></span><span class="content" style="border-bottom: 2px solid RGBA(79, 177, 249, .65); color: #2b2b2b; padding-bottom: 2px;"><span style="width: 30px; height: 30px; display: block; background-image: url(https://imgkr.cn-bj.ufileos.com/cdf294d0-6361-4af9-85e2-0913f0eb609b.png); background-position: center; background-size: 30px; margin: auto; opacity: 1; background-repeat: no-repeat; margin-bottom: -8px;"></span>

19. 10 个线程和2个线程的同步代码，哪个更容易写？
20. 什么是多线程环境下的伪共享（false sharing）？
21. 线程池如何调优，最大数目如何确认？
22. Java 内存模型？
23. 怎么实现所有线程在等待某个事件的发生才会去执行？
24. 说一下 Runnable和 Callable有什么区别？
25. 用Java编程一个会导致死锁的程序，你将怎么解决？
26. 线程的生命周期，线程的几种状态。
27. ReentrantLock实现原理
28. java并发包concurrent及常用的类
29. wait(),notify()和suspend(),resume()之间的区别
30. FutureTask是什么？
31. 一个线程如果出现了运行时异常会怎么样
32. 生产者消费者模型的作用是什么
33. ReadWriteLock是什么
34. Java中用到的线程调度算法是什么？
35. 线程池中的阻塞队列如果满了怎么办？
36. 线程池中 submit()和 execute()方法有什么区别？
37. 介绍一下 AtomicInteger 类的原理？
38. 多线程锁的升级原理是什么？
39. 指令重排序，内存栅栏等？
40. Java 内存模型 happens-before原则
41. 公平锁/非公平锁
42. 可重入锁
43. 独享锁、共享锁
44. 偏向锁/轻量级锁/重量级锁
45. 如何保证内存可见性
46. 非核心线程延迟死亡，如何实现？
47. ConcurrentHashMap读操作为什么不需要加锁？
48. ThreadLocal 如何解决 Hash 冲突？
49. ThreadLocal 的内存泄露是怎么回事？
50. 为什么ThreadLocalMap 的 key是弱引用，设计理念是？
51. 同步方法和同步代码块的区别是什么？
52. 在Java中Lock接口比synchronized块的优势是什么？如果你需要实现一个高效的缓存，它允许多个用户读，但只允许一个用户写，以此来保持它的完整性，你会怎样去实现它？
53. 用Java实现阻塞队列。
54. 用Java写代码来解决生产者——消费者问题。
55. 什么是竞争条件？你怎样发现和解决竞争？
56. 为什么我们调用start()方法时会执行run()方法，为什么我们不能直接调用run()方法？
57. Java中你怎样唤醒一个阻塞的线程？
58. 什么是不可变对象，它对写并发应用有什么帮助？
59. 你在多线程环境中遇到的共同的问题是什么？你是怎么解决它的？
60. Java 中能创建 volatile数组吗
61. volatile 能使得一个非原子操作变成原子操作吗
62. 你是如何调用 wait（）方法的？使用 if 块还是循环？为什么？
63. 我们能创建一个包含可变对象的不可变对象吗？
64. 在多线程环境下，SimpleDateFormat是线程安全的吗
65. 为什么Java中 wait 方法需要在 synchronized 的方法中调用？
66. BlockingQueue，CountDownLatch及Semeaphore的使用场景
67. Java中interrupted 和 isInterruptedd方法的区别？
68. 怎么检测一个线程是否持有对象监视器
69. 什么情况会导致线程阻塞
70. 如何在两个线程间共享数据
71. Thread.sleep(1000)的作用是什么？
72. 使用多线程可能带来什么问题
73. 说说线程的生命周期和状态?
74. 什么是上下文切换
75. Java Monitor 的工作机理
76. 按线程池内部机制，当提交新任务时，有哪些异常要考虑。
77. 线程池都有哪几种工作队列？
78. 说说几种常见的线程池及使用场景?
79. 使用无界队列的线程池会导致内存飙升吗？
80. 为什么阿里发布的 Java开发手册中强制线程池不允许使用 Executors 去创建？
81. Future有缺陷嘛？
