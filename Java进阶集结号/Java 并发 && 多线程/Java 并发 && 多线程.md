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
> 全：（1）对变量的写操作不依赖于当前值（比如 i++），或者说是单纯的变量赋值（boolean
> flag = true）。（2）该变量没有包含在具有其他变量的不变式中， 也就是说，不同的 volatile 变量之间，不能互相依赖。 只有在状态真正独立于程序内其他内容时才能使用 volatile。

> 实现原理：volatile的实现原理是在执行变量写操作后指令lock指令，这个指令会将变量实时写入内存而不是处理器的内存缓冲区，然后其他处理器通过缓存一致性协议嗅探到这个变量的变更，将该变量的缓存设为失效，从而实现内存可见性；
>

> 在JMM中，通过内存屏障实现具体如下：
>

> - 在每个volatile写之前插入一个storestore屏障，会防止volatile写入操作和上面的其他写入操作重排序；

> - 在volatile写入之后插入一个storeload内存屏障，防止上面volatile写操作和下面的读操作重排序；

> - 在volatile读操作之后插入loadload和loadstore内存屏障，防止上面的volatile读操作和下面的普通读操作，volatile写操作和普通写操作重排序；

> 从而实现有序性
> 还有根据happens-before原则，每个volatile写操作先行发生与后续对这个变量的读操作。

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
> 线程太多会占用内存，而且频繁的线程上下文切换也会导致程序运行效率降低。
#### 11. 说说 Semaphore原理？ ####



#### 12. AQS组件，实现原理 ####
AQS，即AbstractQueuedSynchronizer，是构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。可以回答以下这几个关键点哈：
- state 状态的维护。
- CLH队列
- ConditionObject通知
- 模板方法设计模式
- 独占与共享模式。
- 自定义同步器。
- AQS全家桶的一些延伸，如：ReentrantLock等。
- 

**state 状态的维护**

- state，int变量，锁的状态，用volatile修饰，保证多线程中的可见性。
- getState()和setState()方法采用final修饰，限制AQS的子类重写它们两。
- compareAndSetState（）方法采用乐观锁思想的CAS算法操作确保线程安全,保证状态
设置的原子性。

对CAS有兴趣的朋友，可以看下我这篇文章哈~
[CAS乐观锁解决并发问题的一次实践](https://juejin.im/post/6844903869340712967#comment)

**CLH队列**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f37b908ad9b482fb60de6478817a7dc~tplv-k3u1fbpfcp-zoom-1.image)

> **CLH(Craig, Landin, and Hagersten locks) 同步队列** 是一个FIFO双向队列，其内部通过节点head和tail记录队首和队尾元素，队列元素的类型为Node。AQS依赖它来完成同步状态state的管理，当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。

**ConditionObject通知**

我们都知道，synchronized控制同步的时候，可以配合Object的wait()、notify()，notifyAll() 系列方法可以实现等待/通知模式。而Lock呢？它提供了条件Condition接口，配合await(),signal(),signalAll() 等方法也可以实现等待/通知机制。ConditionObject实现了Condition接口，给AQS提供条件变量的支持 。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/385e31246e8c4e1e8e8a9dacb183da74~tplv-k3u1fbpfcp-zoom-1.image)

ConditionObject队列与CLH队列的爱恨情仇：

- 调用了await()方法的线程，会被加入到conditionObject等待队列中，并且唤醒CLH队列中head节点的下一个节点。
- 线程在某个ConditionObject对象上调用了singnal()方法后，等待队列中的firstWaiter会被加入到AQS的CLH队列中，等待被唤醒。
- 当线程调用unLock()方法释放锁时，CLH队列中的head节点的下一个节点(在本例中是firtWaiter)，会被唤醒。

**模板方法设计模式**

什么是模板设计模式？
> 在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤。

AQS的典型设计模式就是模板方法设计模式啦。AQS全家桶（ReentrantLock，Semaphore）的衍生实现，就体现出这个设计模式。如AQS提供tryAcquire，tryAcquireShared等模板方法，给子类实现自定义的同步器。

**独占与共享模式**

- 独占式: 同一时刻仅有一个线程持有同步状态，如ReentrantLock。又可分为公平锁和非公平锁。
- 共享模式:多个线程可同时执行，如Semaphore/CountDownLatch等都是共享式的产物。

**自定义同步器**

你要实现自定义锁的话，首先需要确定你要实现的是独占锁还是共享锁，定义原子变量state的含义，再定义一个内部类去继承AQS，重写对应的模板方法即可啦

**AQS全家桶的一些延伸。**

Semaphore，CountDownLatch，ReentrantLock

可以看下之前我这篇文章哈，[AQS解析与实战](https://juejin.im/post/6844903903188746247)

#### 13. 假设有T1、T2、T3三个线程，你怎样保证T2在T1执行完后执行，T3在T2执行完后执行？ ####


可以使用**join方法**解决这个问题。比如在线程A中，调用线程B的join方法表示的意思就是**：A等待B线程执行完毕后（释放CPU执行权），在继续执行。**

代码如下：
```
public class ThreadTest {

    public static void main(String[] args) {

        Thread spring = new Thread(new SeasonThreadTask("春天"));
        Thread summer = new Thread(new SeasonThreadTask("夏天"));
        Thread autumn = new Thread(new SeasonThreadTask("秋天"));

        try
        {
            //春天线程先启动
            spring.start();
            //主线程等待线程spring执行完，再往下执行
            spring.join();
            //夏天线程再启动
            summer.start();
            //主线程等待线程summer执行完，再往下执行
            summer.join();
            //秋天线程最后启动
            autumn.start();
            //主线程等待线程autumn执行完，再往下执行
            autumn.join();
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        }
    }
}

class SeasonThreadTask implements Runnable{

    private String name;

    public SeasonThreadTask(String name){
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 1; i <4; i++) {
            System.out.println(this.name + "来了: " + i + "次");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```

运行结果：
```
春天来了: 1次
春天来了: 2次
春天来了: 3次
夏天来了: 1次
夏天来了: 2次
夏天来了: 3次
秋天来了: 1次
秋天来了: 2次
秋天来了: 3次
```

#### 14. LockSupport作用是？ ####

- LockSupport作用
- park和unpark，与wait，notify的区别
- Object blocker作用？

LockSupport是个工具类，它的主要作用是挂起和唤醒线程， 该工具类是创建锁和其他同步类的基础。

```
public static void park(); //挂起当前线程，调用unpark(Thread thread)或者当前线程被中断，才能从park方法返回
public static void parkNanos(Object blocker, long nanos);  // 挂起当前线程，有超时时间的限制
public static void parkUntil(Object blocker, long deadline); // 挂起当前线程，直到某个时间
public static void park(Object blocker); //挂起当前线程
public static void unpark(Thread thread); // 唤醒当前thread线程
```

看个例子吧：
```
public class LockSupportTest {

    public static void main(String[] args) {

        CarThread carThread = new CarThread();
        carThread.setName("劳斯劳斯");
        carThread.start();

        try {
            Thread.currentThread().sleep(2000);
            carThread.park();
            Thread.currentThread().sleep(2000);
            carThread.unPark();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class CarThread extends Thread{

        private boolean isStop = false;

        @Override
        public void run() {

            System.out.println(this.getName() + "正在行驶中");

            while (true) {

                if (isStop) {
                    System.out.println(this.getName() + "车停下来了");
                    LockSupport.park(); //挂起当前线程
                }
                System.out.println(this.getName() + "车还在正常跑");

                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }

        public void park() {
            isStop = true;
            System.out.println("停车啦，检查酒驾");

        }

        public void unPark(){
            isStop = false;
            LockSupport.unpark(this); //唤醒当前线程
            System.out.println("老哥你没酒驾，继续开吧");
        }

    }
}

```
运行结果：
```
劳斯劳斯正在行驶中
劳斯劳斯车还在正常跑
劳斯劳斯车还在正常跑
停车啦，检查酒驾
劳斯劳斯车停下来了
老哥你没酒驾，继续开吧
劳斯劳斯车还在正常跑
劳斯劳斯车还在正常跑
劳斯劳斯车还在正常跑
劳斯劳斯车还在正常跑
劳斯劳斯车还在正常跑
劳斯劳斯车还在正常跑
```

LockSupport的park和unpark的实现，有点类似wait和notify的功能。但是
> - park不需要获取对象锁
> - 中断的时候park不会抛出InterruptedException异常，需要在park之后自行判断中断状态
> - 使用park和unpark的时候，可以不用担心park的时序问题造成死锁
> - LockSupport不需要在同步代码块里
> - unpark却可以唤醒一个指定的线程，notify只能随机选择一个线程唤醒

Object blocker作用？
> 方便在线程dump的时候看到具体的阻塞对象的信息。

#### 15. Condition接口及其实现原理 ####
- Condition接口与Object监视器方法对比
- Condition接口使用demo
- Condition实现原理

**Condition接口与Object监视器方法对比**

Java对象（Object），提供wait()、notify()，notifyAll() 系列方法，配合synchronized，可以实现等待/通知模式。而Condition接口配合Lock，通过await(),signal(),signalAll() 等方法，也可以实现类似的等待/通知机制。

| 对比项 | 对象监视方法| Condition |
|-----|-----|------|
| 前置条件 | 获得对象的锁  | 调用Lock.lock()获取锁,调用Lock.newCondition()获得Condition对象|
| 调用方式 | 直接调用，object.wait()  | 直接调用，condition.await() |
| 等待队列数 | 1个   | 多个 |
| 当前线程释放锁并进入等待状态 | 支持  | 支持 |
| 在等待状态中不响应中断 | 不支持  | 支持 |
| 当前线程释放锁并进入超时等待状态| 支持  | 支持 |
| 当前线程释放锁并进入等待状态到将来的某个时间| 不支持  | 支持 |
| 唤醒等待队列中的一个线程| 支持  | 支持 |
| 唤醒等待队列中的全部线程| 支持  | 支持 |

**Condition接口使用demo**

```
public class ConditionTest {
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    public void conditionWait() throws InterruptedException {
        lock.lock();
        try {
            condition.await();
        } finally {
            lock.unlock();
        }
    }

    public void conditionSignal() throws InterruptedException {
        lock.lock();
        try {
            condition.signal();
        } finally {
            lock.unlock();
        }
    }
}

```
**Condition实现原理**

其实，同步队列和等待队列中节点类型都是同步器的静态内部类 AbstractQueuedSynchronizer.Node，接下来我们图解一下Condition的实现原理~

**等待队列的基本结构图**
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6f62d7c11ea4907b84924c4a02cee7f~tplv-k3u1fbpfcp-zoom-1.image)
> 一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点 （lastWaiter）。当前线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队

**AQS 结构图**

ConditionI是跟Lock一起结合使用的，底层跟同步器（AQS）相关。同步器拥有一个同步队列和多个等待队列~
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f342c548da8c42d6a60a0c19aeee8489~tplv-k3u1fbpfcp-zoom-1.image)


**等待**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/677bf2a7edd8447b9f21e626e6667aa3~tplv-k3u1fbpfcp-zoom-1.image)
>  当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。


**通知**

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bac0705fa308413b97eed7a9d8b938c6~tplv-k3u1fbpfcp-zoom-1.image)
> 调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在
唤醒节点之前，会将节点移到同步队列中。

#### 16. 说说并发与并行的区别? ####

> 并发是指两个或以上的任务在同一时间段内同时执行，由于操作系统CPU时间分片功能，让我们看起来任务是在同时执行，但实际上还是串行执行；如果并发任务运行在多核CPU的硬件上，任务会有部分时间片段存在并行执行。
> 
> 并行是指两个或以上的任务在同一时刻同时执行。例如：两个线程同时运行在两个单核的CPU上，那么这两个线程就是并行执行的。

#### 17. 为什么要用线程池？Java的线程池内部机制，参数作用，几种工作阻塞队列，线程池类型以及使用场景 ####

回答这些点：
- 为什么要用线程池？
- Java的线程池原理
- 线程池核心参数
- 几种工作阻塞队列
- 线程池使用不当的问题
- 线程池类型以及使用场景

**为什么要用线程池？**

线程池：一个管理线程的池子。
- 管理线程，避免增加创建线程和销毁线程的资源损耗。
- 提高响应速度。
- 重复利用。

**Java的线程池执行原理**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/efe9ed82093e4c8bab768eac79dffed3~tplv-k3u1fbpfcp-zoom-1.image)
为了形象描述线程池执行，打个比喻：
- 核心线程比作公司正式员工
- 非核心线程比作外包员工
- 阻塞队列比作需求池
- 提交任务比作提需求
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ed3df3db91941e9b8d3e1078fdd02b5~tplv-k3u1fbpfcp-zoom-1.image)

**线程池核心参数**

```
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize,
   long keepAliveTime,
   TimeUnit unit,
   BlockingQueue<Runnable> workQueue,
   ThreadFactory threadFactory,
   RejectedExecutionHandler handler) 
```
- corePoolSize： 线程池核心线程数最大值
- maximumPoolSize： 线程池最大线程数大小
- keepAliveTime： 线程池中非核心线程空闲的存活时间大小
- unit： 线程空闲存活时间单位
- workQueue： 存放任务的阻塞队列
- threadFactory： 用于设置创建线程的工厂，可以给创建的线程设置有意义的名字，可方便排查问题。
- handler：线城池的饱和策略事件，主要有四种类型拒绝策略。

**四种拒绝策略**

- AbortPolicy(抛出一个异常，默认的)
- DiscardPolicy(直接丢弃任务)
- DiscardOldestPolicy（丢弃队列里最老的任务，将当前这个任务继续提交给线程池）
- CallerRunsPolicy（交给线程池调用所在的线程进行处理)

**几种工作阻塞队列**

- ArrayBlockingQueue（用数组实现的有界阻塞队列，按FIFO排序量）
- LinkedBlockingQueue（基于链表结构的阻塞队列，按FIFO排序任务，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列）
- DelayQueue（一个任务定时周期的延迟执行的队列）
- PriorityBlockingQueue（具有优先级的无界阻塞队列）
- SynchronousQueue（一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态）

**线程池使用不当的问题**

线程池适用不当可能导致内存飙升问题哦

有兴趣可以看我这篇文章哈:[源码角度分析-newFixedThreadPool线程池导致的内存飙升问题](https://juejin.im/post/6844903930502070285)

**线程池类型以及使用场景**

- newFixedThreadPool
> 适用于处理CPU密集型的任务，确保CPU在长期被工作线程使用的情况下，尽可能的少的分配线程，即适用执行长期的任务。
- newCachedThreadPool
> 用于并发执行大量短期的小任务。
- newSingleThreadExecutor
> 适用于串行执行任务的场景，一个任务一个任务地执行。
- newScheduledThreadPool
> 周期性执行任务的场景，需要限制线程数量的场景
- newWorkStealingPool 
> 建一个含有足够多线程的线程池，来维持相应的并行级别，它会通过工作窃取的方式，使得多核的 CPU 不会闲置，总会有活着的线程让 CPU 去运行,本质上就是一个 ForkJoinPool。)


有兴趣可以看我这篇文章哈:[面试必备：Java线程池解析](https://juejin.im/post/6844903889678893063)

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

#### 19. 10 个线程和2个线程的同步代码，哪个更容易写？ ####

> 从写代码的角度来说，两者的复杂度是相同的，因为同步代码与线程数量是相互独立的。但是同步策略的选择依赖于线程的数量，因为越多的线程意味着更大的竞争，所以你需要利用同步技术，如锁分离，这要求更复杂的代码和专业知识。

#### 20. 什么是多线程环境下的伪共享（false sharing）？ ####
- 什么是伪共享
- 如何解决伪共享问题

**什么是伪共享**

伪共享定义？
> CPU的缓存是以缓存行(cache line)为单位进行缓存的，当多个线程修改相互独立的变量，而这些变量又处于同一个缓存行时就会影响彼此的性能。这就是伪共享

现代计算机计算模型，大家都有印象吧？我之前这篇文章也有讲过，有兴趣可以看一下哈，[Java程序员面试必备：Volatile全方位解析](https://juejin.im/post/6859390417314512909)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3bcae0d0fb44b1b9fb2db5093e6dd5d~tplv-k3u1fbpfcp-zoom-1.image)
- CPU执行速度比内存速度快好几个数量级，为了提高执行效率，现代计算机模型演变出CPU、缓存（L1，L2，L3），内存的模型。
- CPU执行运算时，如先从L1缓存查询数据，找不到再去L2缓存找，依次类推，直到在内存获取到数据。
- 为了避免频繁从内存获取数据，聪明的科学家设计出缓存行，缓存行大小为64字节。

也正是因为缓存行，就导致伪共享问题的存在，如图所示：
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/886ab0227a174842a2976581472eec06~tplv-k3u1fbpfcp-zoom-1.image)

假设数据a、b被加载到同一个缓存行。
- 当线程1修改了a的值，这时候CPU1就会通知其他CPU核，当前缓存行（Cache line）已经失效。
- 这时候，如果线程2发起修改b，因为缓存行已经失效了，所以**core2 这时会重新从主内存中读取该 Cache line 数据**。读完后，因为它要修改b的值，那么CPU2就通知其他CPU核，当前缓存行（Cache line）又已经失效。
- 酱紫，如果同一个Cache line的内容被多个线程读写，就很容易产生相互竞争，频繁回写主内存，会大大降低性能。

**如何解决伪共享问题**

既然伪共享是因为相互独立的变量存储到相同的Cache line导致的，一个缓存行大小是64字节。那么，我们就可以**使用空间换时间**，即数据填充的方式，把独立的变量分散到不同的Cache line~

共享内存demo例子:
```
public class FalseShareTest  {

    public static void main(String[] args) throws InterruptedException {
        Rectangle rectangle = new Rectangle();
        long beginTime = System.currentTimeMillis();
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 100000000; i++) {
                rectangle.a = rectangle.a + 1;
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 100000000; i++) {
                rectangle.b = rectangle.b + 1;
            }
        });

        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();

        System.out.println("执行时间" + (System.currentTimeMillis() - beginTime));
    }

}

class Rectangle {
    volatile long a;
    volatile long b;
}
```

运行结果：
```
执行时间2815
```
一个long类型是8字节，我们在变量a和b之间不上7个long类型变量呢，输出结果是啥呢？如下：
```
class Rectangle {
    volatile long a;
    long a1,a2,a3,a4,a5,a6,a7;
    volatile long b;
}
```
运行结果：
```
执行时间1113
```
可以发现利用填充数据的方式，让读写的变量分割到不同缓存行，可以很好挺高性能~

#### 21. 线程池如何调优，最大数目如何确认？ ####

在《Java Concurrency in Practice》一书中，有一个评估线程池线程大小的公式
>  **Nthreads=Ncpu*Ucpu*(1+w/c)**
>
> - Ncpu = CPU总核数
- Ucpu =cpu使用率，0~1
- W/C=等待时间与计算时间的比率

假设cpu 100%运转，则公式为
```
Nthreads=Ncpu*(1+w/c)
```

**估算的话，酱紫：**
- 如果是**IO密集型应用**（如数据库数据交互、文件上传下载、网络数据传输等等），IO操作一般比较耗时，等待时间与计算时间的比率（w/c）会大于1，所以最佳线程数估计就是 Nthreads=Ncpu*（1+1）= 2Ncpu 。
- 如果是**CPU密集型应用**（如算法比较复杂的程序），最理想的情况，没有等待，w=0，Nthreads=Ncpu。又对于计算密集型的任务，在拥有N个处理器的系统上，当线程池的大小为N+1时，通常能实现最优的效率。所以 Nthreads = Ncpu+1

有具体指参考呢？举个例子
> 比如平均每个线程CPU运行时间为0.5s，而线程等待时间（非CPU运行时间，比如IO）为1.5s，CPU核心数为8，那么根据上面这个公式估算得到：线程池大小=(1+1.5/05)*8 =32。

参考了网上这篇文章，写得很棒，有兴趣的朋友可以去看一下哈：
- [根据CPU核心数确定线程池并发线程数](https://www.cnblogs.com/dennyzhangdd/p/6909771.html)

#### 22. Java 内存模型？ ####

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/275f5b038d1d4e9ba308ab129df4aef3~tplv-k3u1fbpfcp-zoom-1.image)

#### 23. 怎么实现所有线程在等待某个事件的发生才会去执行？ ####

> 方案一：读写锁
> 
　　刚开始主线程先获取写锁，然后所有子线程获取读锁，然后等事件发生时主线程释放写锁；

> 方案二：CountDownLatch
> 
　　CountDownLatch初始值设为1，所有子线程调用await方法等待，等事件发生时调用countDown方法计数减为0；

> 方案三：Semaphore
> 
　　Semaphore初始值设为N，刚开始主线程先调用acquire(N)申请N个信号量，其它线程调用acquire()阻塞等待，等事件发生时同时主线程释放N个信号量；

#### 24. 说一下 Runnable和 Callable有什么区别？ ####
- Callable接口方法是call()，Runnable的方法是run()；
- Callable接口call方法有返回值，支持泛型，Runnable接口run方法无返回值。
- Callable接口call()方法允许抛出异常；而Runnable接口run()方法不能继续上抛异常；

```
@FunctionalInterface
public interface Callable<V> {
    /**
     * 支持泛型V，有返回值，允许抛出异常
     */
    V call() throws Exception;
}

@FunctionalInterface
public interface Runnable {
    /**
     *  没有返回值，不能继续上抛异常
     */
    public abstract void run();
}

```

看下demo代码吧，这样应该好理解一点哈~
```
/*
 *  @Author 捡田螺的小男孩
 *  @date 2020-08-18
 */
public class CallableRunnableTest {

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(5);

        Callable <String> callable =new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "你好，callable";
            }
        };

        //支持泛型
        Future<String> futureCallable = executorService.submit(callable);

        try {
            System.out.println(futureCallable.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("你好呀,runnable");
            }
        };

        Future<?> futureRunnable = executorService.submit(runnable);
        try {
            System.out.println(futureRunnable.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        executorService.shutdown();

    }
}
```
运行结果：
```
你好，callable
你好呀,runnable
null
```

#### 25. 用Java编程一个会导致死锁的程序，你将怎么解决？ ####

	public class DeadLock {
	    public static final String LOCK_1 = "lock1";
	    public static final String LOCK_2 = "lock2";
	
	    public static void main(String[] args) {
	        Thread threadA = new Thread(() -> {
	            try {
	                while (true) {
	                    synchronized (DeadLock.LOCK_1) {
	                        System.out.println(Thread.currentThread().getName() + " 锁住 lock1");
	                        Thread.sleep(1000);
	                        synchronized (DeadLock.LOCK_2) {
	                            System.out.println(Thread.currentThread().getName() + " 锁住 lock2");
	                        }
	                    }
	                }
	            } catch (Exception e) {
	                e.printStackTrace();
	            }
	        });
	
	        Thread threadB = new Thread(() -> {
	            try {
	                while (true) {
	                    synchronized (DeadLock.LOCK_2) {
	                        System.out.println(Thread.currentThread().getName() + " 锁住 lock2");
	                        Thread.sleep(1000);
	                        synchronized (DeadLock.LOCK_1) {
	                            System.out.println(Thread.currentThread().getName() + " 锁住 lock1");
	                        }
	                    }
	                }
	            } catch (Exception e) {
	                e.printStackTrace();
	            }
	        });
	
	        threadA.start();
	        threadB.start();
	    }
	}
> 想要解决这个死锁很简单，我们只需要让threadA和threadB获取DeadLock.LOCK_1和DeadLock.LOCK_2的顺序相同即可，例如：

	public class DeadLock {
	    public static final String LOCK_1 = "lock1";
	    public static final String LOCK_2 = "lock2";
	
	    public static void main(String[] args) {
	        Thread threadA = new Thread(() -> {
	            try {
	                while (true) {
	                    synchronized (DeadLock.LOCK_1) {
	                        System.out.println(Thread.currentThread().getName() + " 锁住 lock1");
	                        Thread.sleep(1000);
	                        synchronized (DeadLock.LOCK_2) {
	                            System.out.println(Thread.currentThread().getName() + " 锁住 lock2");
	                        }
	                    }
	                }
	            } catch (Exception e) {
	                e.printStackTrace();
	            }
	        });
	
	        Thread threadB = new Thread(() -> {
	            try {
	                while (true) {
	                    synchronized (DeadLock.LOCK_1) {
	                        System.out.println(Thread.currentThread().getName() + " 锁住 lock1");
	                        Thread.sleep(1000);
	                        synchronized (DeadLock.LOCK_2) {
	                            System.out.println(Thread.currentThread().getName() + " 锁住 lock2");
	                        }
	                    }
	                }
	            } catch (Exception e) {
	                e.printStackTrace();
	            }
	        });
	
	        threadA.start();
	        threadB.start();
	    }
	}
> 除此之外，还有一种解决方法，那就是让DeadLock.LOCK_1和DeadLock.LOCK_2的值相同，例如：
> 
    public static final String LOCK_1 = "lock";
    public static final String LOCK_2 = "lock";
> 这是为什么呢？因为字符串有一个常量池，如果不同的线程持有的锁是具有相同字符的字符串锁时，那么两个锁实际上就是同一个锁。

#### 26. 线程的生命周期，线程的几种状态。 ####

> 新建状态（new）；就绪状态（start）；运行状态（该线程会进入Running执行状态，并且开始调用run()方法中逻辑代码）；阻塞状态（wait）；销毁状态（destroy）；
> ![](https://img2018.cnblogs.com/blog/1223046/201907/1223046-20190722214114154-276488899.png)

#### 27. ReentrantLock实现原理 ####



#### 28. java并发包concurrent及常用的类 ####

> Condition.java;CountDownLatch.java;CyclicBarrier.java;Semaphore.java;ReentrantReadWriteLock.java;Callable.java;线程池：提供的线程池有几种：

> //有数量限制的线程池
ExecutorService service=Executors.newFixedThreadPool(4);
//没有数量限制的线程池
ExecutorService service=Executors.newCachedThreadPool();
//单线程池
ExecutorService service=Executors.newSingleThreadExecutor();
他们都是通过下面这个线程池实现的
有数量线程池的实现方式

> 	public static ExecutorService newFixedThreadPool(int nThreads) {
> 	return new ThreadPoolExecutor(nThreads/*核心线程数*/, nThreads/*最高线程数*/,
> 	                                      0L/*高出核心线程数的线程最高存活时间*/, TimeUnit.MILLISECONDS/*高出核心线程数的线程最高存活时间单位*/,
> 	                                      new LinkedBlockingQueue<Runnable>()/*任务队列*/);
> 	
> 	}
>

#### 29. wait(),notify()和suspend(),resume()之间的区别 ####

- > wait() 使得线程进入阻塞等待状态，并且释放锁
- > notify()唤醒一个处于等待状态的线程，它一般跟wait（）方法配套使用。
- > suspend()使得线程进入阻塞状态，并且不会自动恢复，必须对应的resume() 被调用，才能使得线程重新进入可执行状态。suspend()方法很容易引起死锁问题。
- > resume()方法跟suspend()方法配套使用。

> suspend()不建议使用,suspend()方法在调用后，线程不会释放已经占有的资 源（比如锁），而是占有着资源进入睡眠状态，这样容易引发死锁问题。
>

#### 30. FutureTask是什么？ ####

> FutureTask实现了RunnableFuture接口，RunnableFuture接口又实现了Runnable，Future接口。该类提供了Future的基本实现（Future表示异步计算的结果），并返回异步执行的计算结果。它提供了一些Future方法的实现，可以检查计算是否完成（isDone），get()方法获取计算结果，如果计算尚未完成则将方法阻塞（阻塞时间可以指定）。cancel()方法取消任务执行，提供了isCancelled()方法来确定任务是正常完成还是被取消。一旦计算完成，就不能取消计算。如果您想使用Future是为了取消任务而不需要提供返回的结果，则可以使生命返回值为Future类型，并由于基础任务而返回null。

#### 31. 一个线程如果出现了运行时异常会怎么样 ####

> Java中Throwable分为Exception和Error：
出现Error的情况下，程序会停止运行。
Exception分为RuntimeException和非运行时异常。
非运行时异常必须处理，比如thread中sleep()时，必须处理InterruptedException异常，才能通过编译。
而RuntimeException可以处理也可以不处理，因为编译并不能检测该类异常，比如NullPointerException、ArithmeticException）和 ArrayIndexOutOfBoundException等。

> 由此题目所诉情形下发生的应该是RuntimeException，属于未检测异常，编译器不会检查该异常，可以处理，也可不处理。
所以这里存在两种情形：

> ① 如果该异常被捕获或抛出，则程序继续运行。

> ② 如果异常没有被捕获该线程将会停止执行。

> Thread.UncaughtExceptionHandler是用于处理未捕获异常造成线程突然中断情况的一个内嵌接口。当一个未捕获异常将造成线程中断的时候JVM会使用Thread.getUncaughtExceptionHandler()来查询线程的UncaughtExceptionHandler，并将线程和异常作为参数传递给handler的uncaughtException()方法进行处理。

#### 32. 生产者消费者模型的作用是什么 ####

> （1）通过平衡生产能力和消费能力来提升整个系统的运行效率。
>
> （2）解耦。生产者和消费者互相不受影响，可独立开发而不受对方制约。

#### 33.ReadWriteLock是什么

#### 34.Java中用到的线程调度算法是什么？

> java虚拟机使用抢占式线程调度模型。即当一个线程用完CPU之后，操作系统会根据线程优先级、线程饥饿情况的数据计算出一个总的线程优先级并分配下一个时间片给优先级高的线程。
>
> 操作系统中可能会出现某条线程常常获取到CPU控制权的情况，为了让某些优先级比较低的线程也能获取到CPU的控制权，可以使用Thread.sleep(0)手动触发一次操作系统分配时间片的操作，这也是平衡CPU控制权的一种操作。

#### 35.线程池中的阻塞队列如果满了怎么办？

> 线程池如果使用无界阻塞队列，在新的任务不断提交到线程池时，回事队列变得越来越大，此时会导致内存飙升进而出现OOM异常。解决办法是给线程池设置有界队列加一个拒绝策略。
>
> 四种拒绝策略：ThreadPoolExecutor.AbortPolicy（丢弃任务并抛出RejectedExecutionException异常，线程池默认策略）；ThreadPoolExecutor.DiscardPolicy（直接丢弃任务但不抛出异常）；ThreadPoolExecutor.DiscardOldestPolicy（丢弃队列中老的任务，然后重新尝试执行任务，不能执行任务就再次丢弃老的任务，直到可执行任务为止）；ThreadPoolExecutor.CallerRunsPolicy（交由调用线程执行任务，即主线程处理该任务）；
>
> 生产环境线程池队列满了又不能丢弃任务的解决方案：可以根据自己的业务需要，选择通过实现RejectedExecutionHandler接口来自定义拒绝策略。比如把线程池无法执行的任务信息持久化到数据库，后台专门起一个线程，后续等待线程池的工作负载降低了，这个后台线程就可以慢慢的从磁盘里读取之前持久化的任务重新提交到线程池。

#### 36.线程池中 submit()和 execute()方法有什么区别？

> （1）execute()只能执行实现了Runnable接口的线程，submit()可以执行实现了Runnable接口或Callable接口的线程。
>

> （2）execute()方法没有返回值，而submit()方法有返回值。
>
> （3）submit()是ExecutorService接口中定义的方法，execute()是ThreadPoolExecutor中定义的方法；

#### 37.介绍一下 AtomicInteger 类实现原子操作的原理？

> 我们先来看一下getAndIncrement的源代码：
>
> ```
> public final int getAndIncrement() {
>        for (;;) {
>              int current = get();  // 取得AtomicInteger里存储的数值
>            int next = current + 1;  // 加1
>            if (compareAndSet(current, next))   // 调用compareAndSet执行原子更新操作
>                return current;
>        }
>    }
> ```
>
> 这段代码写的很巧妙：
> （1）compareAndSet方法首先判断当前值是否等于current；
> （2）如果当前值 = current ，说明AtomicInteger的值没有被其他线程修改；
> （3）如果当前值 != current，说明AtomicInteger的值被其他线程修改了，这时会再次进入循环重新比较；
>
> 注意这里的compareAndSet方法，源代码如下：
>
> ```
> public final boolean compareAndSet(int expect, int update) {
>     return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
> }
> ```
>
> 调用Unsafe来实现
> private static final Unsafe unsafe = Unsafe.getUnsafe();

#### 37.多线程锁的升级原理是什么？

> 锁的级别从低到高：
>
> 无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁
>
>  
>
> 锁分级别原因：
>
> 没有优化以前，synchronized是重量级锁（悲观锁），使用 wait 和 notify、notifyAll 来切换线程状态非常消耗系统资源；线程的挂起和唤醒间隔很短暂，这样很浪费资源，影响性能。所以 JVM 对 synchronized 关键字进行了优化，把锁分为 无锁、偏向锁、轻量级锁、重量级锁 状态。
>
>  
>
> 无锁：没有对资源进行锁定，所有的线程都能访问并修改同一个资源，但同时只有一个线程能修改成功，其他修改失败的线程会不断重试直到修改成功。
>
>  
>
> 偏向锁：对象的代码一直被同一线程执行，不存在多个线程竞争，该线程在后续的执行中自动获取锁，降低获取锁带来的性能开销。偏向锁，指的就是偏向第一个加锁线程，该线程是不会主动释放偏向锁的，只有当其他线程尝试竞争偏向锁才会被释放。
>
> 偏向锁的撤销，需要在某个时间点上没有字节码正在执行时，先暂停拥有偏向锁的线程，然后判断锁对象是否处于被锁定状态。如果线程不处于活动状态，则将对象头设置成无锁状态，并撤销偏向锁；
>
> 如果线程处于活动状态，升级为轻量级锁的状态。
>
>  
>
> 轻量级锁：轻量级锁是指当锁是偏向锁的时候，被第二个线程 B 所访问，此时偏向锁就会升级为轻量级锁，线程 B 会通过自旋的形式尝试获取锁，线程不会阻塞，从而提高性能。
>
> 当前只有一个等待线程，则该线程将通过自旋进行等待。但是当自旋超过一定的次数时，轻量级锁便会升级为重量级锁；当一个线程已持有锁，另一个线程在自旋，而此时又有第三个线程来访时，轻量级锁也会升级为重量级锁。
>
>  
>
> 重量级锁：指当有一个线程获取锁之后，其余所有等待获取该锁的线程都会处于阻塞状态。
>
> 重量级锁通过对象内部的监视器（monitor）实现，而其中 monitor 的本质是依赖于底层操作系统的 Mutex Lock 实现，操作系统实现线程之间的切换需要从用户态切换到内核态，切换成本非常高。
>
> ![1619622902332](C:\Users\Administrator.USER-20190223MK\AppData\Roaming\Typora\typora-user-images\1619622902332.png)



#### 38.指令重排序，内存栅栏等？

> 见题7
>

#### 39.Java 内存模型 happens-before原则

> jvm专题40题
>

#### 40.公平锁/非公平锁

> 公平锁就是先进这个队列的线程，也先出队列获得资源执行任务，而非公平锁的话，则是线程还没有进队列之前可以与队列中的线程竞争尝试获得锁，如果获取失败，则进队列，此时也是要乖乖等前面出队才行。 
>

> 公平锁是在ReentrankLock中实现的，synchronized中实现的是非公平锁。
>

#### 41.可重入锁

>  可重入锁就是一个线程可重复获得同一把锁资源。主要是用途就是在递归方面，还有就是防止死锁，比如在一个同步方法块中调用了另一个相同锁对象的同步方法块 。
>

#### 42.独享锁、共享锁

共享锁可以由多个线程获取使用，而独享锁只能由一个线程获取。 对ReentrantReadWriteLock其读锁是共享锁，其写锁是独占锁 读锁的共享锁可保证并发读是非常高效的，读写，写读，写写的过程是互斥的。其中获得写锁的线程还能同时获得读锁，然后通过释放写锁来降级。读锁则不能升级 。**读写锁的升降级？**

#### 43.偏向锁/轻量级锁/重量级锁

见题37

#### 44.如何保证内存可见性

> 见题7

#### 45.非核心线程延迟死亡，如何实现？

>  通过阻塞队列poll()，让线程阻塞等待一段时间，如果没有取到任务，则线程死亡 
>

#### 46.ConcurrentHashMap读操作为什么不需要加锁？

>  在jdk1.7中是采用Segment + HashEntry + ReentrantLock的方式进行实现的，而1.8中放弃了Segment臃肿的设计，取而代之的是采用Node + CAS + Synchronized来保证并发安全进行实现。 
>
> 1. JDK1.8的实现降低锁的粒度，JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）
> 2. JDK1.8版本的数据结构变得更加简单，使得操作也更加清晰流畅，因为已经使用synchronized来进行同步，所以不需要分段锁的概念，也就不需要Segment这种数据结构了，由于粒度的降低，实现的复杂度也增加了
> 3. JDK1.8使用红黑树来优化链表，基于长度很长的链表的遍历是一个很漫长的过程，而红黑树的遍历效率是很快的，代替一定阈值的链表，这样形成一个最佳拍档
>
> ![](https://img-blog.csdnimg.cn/20190802152603746.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2wxODg0ODk1NjczOQ==,size_16,color_FFFFFF,t_70)

> ### get操作源码
>
> 1. 首先计算hash值，定位到该table索引位置，如果是首节点符合就返回
> 2. 如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点，匹配就返回
> 3. 以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null
>
>  get没有加锁的话，ConcurrentHashMap是如何保证读到的数据不是脏数据的呢？ 
>
> ### volatile登场
>
> 对于可见性，Java提供了volatile关键字来保证可见性、有序性。但不保证原子性。
>
> 普通的共享变量不能保证可见性，因为普通共享变量被修改之后，什么时候被写入主存是不确定的，当其他线程去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性。
>
> - volatile关键字对于基本类型的修改可以在随后对多个线程的读保持一致，但是对于引用类型如数组，实体bean，仅仅保证引用的可见性，但并不保证引用内容的可见性。。
> - 禁止进行指令重排序。
>
>  背景：为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存（L1，L2或其他）后再进行操作，但操作完不知道何时会写到内存。 
>
> - 如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。
> - 在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。
>
> **总结下来：**
>
> 第一：使用volatile关键字会强制将修改的值立即写入主存；
>
> 第二：使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量的缓存行无效（反映到硬件层的话，就是CPU的L1或者L2缓存中对应的缓存行无效）；
>
> 第三：由于线程1的工作内存中缓存变量的缓存行无效，所以线程1再次读取变量的值时会去主存读取。
>
> ### 是加在数组上的volatile吗?
>
> ```
>   transient volatile Node<K,V>[] table;
> ```
>
>  我们知道volatile可以修饰数组的，只是意思和它表面上看起来的样子不同。举个栗子，volatile int array[10]是指array的地址是volatile的而不是数组元素的值是volatile的. 
>
> ### 用volatile修饰的Node
>
>  get操作可以无锁是由于Node的元素val和指针next是用volatile修饰的，在多线程环境下线程A修改结点的val或者新增节点的时候是对线程B可见的。 
>
>  既然volatile修饰数组对get操作没有效果那加在数组上的volatile的目的是什么呢？ 
>
>  其实就是为了使得Node数组在扩容的时候对其他线程具有可见性而加的volatile 
>
> ### 总结
>
> - 在1.8中ConcurrentHashMap的get操作全程不需要加锁，这也是它比其他并发集合比如hashtable、用Collections.synchronizedMap()包装的hashmap;安全效率高的原因之一。
> - get操作全程不需要加锁是因为Node的成员val是用volatile修饰的和数组用volatile修饰没有关系。
> - 数组用volatile修饰主要是保证在数组扩容的时候保证可见性。

#### 47.ThreadLocal 如何解决 Hash 冲突？

> 和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用**线性探测**的方式，所谓线性探测，就是根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。
>  ThreadLocalMap解决Hash冲突的方式就是简单的步长加1或减1，寻找下一个相邻的位置。 
>
>  显然ThreadLocalMap采用线性探测的方式解决Hash冲突的***\*效率很低\****，如果有大量不同的ThreadLocal对象放入map中时发送冲突，或者发生二次冲突，则效率很低。 
>
>  **所以这里引出的良好建议是：每个线程只存一个变量，这样的话所有的线程存放到map中的Key都是相同的ThreadLocal，如果一个线程要保存多个变量，就需要创建多个ThreadLocal，多个ThreadLocal放入Map中时会极大的增加Hash冲突的可能。** 

#### 48.ThreadLocal 的内存泄露是怎么回事？

> ThreadLocal在ThreadLocalMap中是以一个弱引用身份被Entry中的Key引用的
>
> 由于Entry的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

#### 49.为什么ThreadLocalMap 的 key是弱引用，设计理念是？

> key 使用强引用：引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。
>
> key 使用弱引用：引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。value在下一次ThreadLocalMap调用set,get，remove的时候会被清除。
>
> 比较两种情况，我们可以发现：由于ThreadLocalMap的生命周期跟Thread一样长，如果都没有手动删除对应key，都会导致内存泄漏，但是使用弱引用可以多一层保障：弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除。
>
> **如何避免泄漏**
> 既然Key是弱引用，那么我们要做的事，就是在调用ThreadLocal的get()、set()方法时完成后再调用remove方法，将Entry节点和Map的引用关系移除，这样整个Entry对象在GC Roots分析后就变成不可达了，下次GC的时候就可以被回收。
>
> 如果使用ThreadLocal的set方法之后，没有显示的调用remove方法，就有可能发生内存泄露，所以养成良好的编程习惯十分重要，使用完ThreadLocal之后，记得调用remove方法。
>
> ```
> ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();
> try {
>     threadLocal.set(new Session(1, "Misout的博客"));
>     // 其它业务逻辑
> } finally {
>     threadLocal.remove();
> ```
>
> - 每个ThreadLocal只能保存一个变量副本，如果想要上线一个线程能够保存多个副本以上，就需要创建多个ThreadLocal。
> - ThreadLocal内部的ThreadLocalMap键为弱引用，会有内存泄漏的风险。

#### 50.同步方法和同步代码块的区别是什么？

> 同步方法默认用this或者当前类class对象作为锁；
>
> 同步代码块可以选择用什么来加锁，比同步方法加锁的粒度更细，我们可以选择只同步发生同步问题的部分代码而不是整个方法；
>
> 同步方法使用synchronized关键字修饰方法，而同步代码块主要修饰需要进行同步的代码块，用synchronized(object) {同步代码块}进行修饰；

#### 51.在Java中Lock接口比synchronized块的优势是什么？如果你需要实现一个高效的缓存，它允许多个用户读，但只允许一个用户写，以此来保持它的完整性，你会怎样去实现它？

> Lock和synchronized有一下几点不同：
>
> （1）Lock是一个接口，而synchronized是java关键字，是内置的语言实现的；
>
> （2）synchronized在发生异常是会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally中释放锁；
>
> （3）Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能等待响应中断；
>
> （4）通过Lock可以知道有没有成功获取锁，而synchronized却无法办到；
>
> （5）Lock可以提高多个线程进行读操作的效率；
>
> 在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时，此时Lock的性能要远远优于synchronized的（竞争激烈的情况下synchronized是重量级锁，所以性能不高）。在使用时要根据实际情况来选择。
>
>  实现高速缓存一般使用读写锁实现，读写锁在读读之间是共享锁，读写、写读、写写之间是互斥锁；使用java.util.concurrent.locks包下面ReadWriteLock接口,该接口下面的实现类[ReentrantReadWriteLock](https://blog.csdn.net/weixin_34362991/article/details/92712655)维护了一个读锁和一个写解锁，具体实现代码示例如下：
>
> ```
> import java.util.Date;
> import java.util.concurrent.locks.ReadWriteLock;
> import java.util.concurrent.locks.ReentrantReadWriteLock;
>  
> /**
>  * 你需要实现一个高效的缓存，它允许多个用户读，但只允许一个用户写，以此来保持它的完整性，你会怎样去实现它？
>  * @author user
>  *
>  */
> public class Test2 {
> 	public static void main(String[] args) {
> 		for (int i = 0; i < 3; i++) {
> 			new Thread(new Runnable() {
> 				
> 				@Override
> 				public void run() {
> 					MyData.read();
> 				}
> 			}).start();
> 		}
> 		for (int i = 0; i < 3; i++) {
> 			new Thread(new Runnable() {
> 				
> 				@Override
> 				public void run() {
> 					MyData.write("a");
> 				}
> 			}).start();
> 		}
> 	}
> }
>  
> class MyData{
> 	//数据
> 	private static String data = "0";
> 	//读写锁
> 	private static ReadWriteLock rw = new ReentrantReadWriteLock();
> 	//读数据
> 	public static void read(){
> 		rw.readLock().lock();
> 		System.out.println(Thread.currentThread()+"读取一次数据："+data+"时间："+new Date());
> 		try {
> 			Thread.sleep(1000);
> 		} catch (InterruptedException e) {
> 			e.printStackTrace();
> 		} finally {
> 			rw.readLock().unlock();
> 		}
> 	}
> 	//写数据
> 	public static void write(String data){
> 		rw.writeLock().lock();
> 		System.out.println(Thread.currentThread()+"对数据进行修改一次："+data+"时间："+new Date());
> 		try {
> 			Thread.sleep(1000);
> 		} catch (InterruptedException e) {
> 			e.printStackTrace();
> 		} finally {
> 			rw.writeLock().unlock();
> 		}
> 	}
> }
> ```

#### 52.用Java实现阻塞队列。

> **方法一：synchronized, wait, notify**
>
> ```
> public class Resource {
>     //当前资源的数量
>     int num = 0;
>     //当前资源的上限
>     int size = 10;
>  
>     //消费资源
>     public synchronized void remove() {
>         //如果num为0，没有资源了，需要等待
>         while (num == 0) {//这里jdk源码里推荐用while，因为有可能出现虚假唤醒，所以要再次确认
>             try {
>                 System.out.println("消费者进入等待");
>                 this.wait();//线程等待，并释放锁
>             } catch (InterruptedException e) {
>                 e.printStackTrace();
>             }
>         }
>         //如果线程可以执行到这里，说明资源里有资源可以消费
>         num--;
>         System.out.println("消费者线程为:" + Thread.currentThread().getName() + "--资源数量:" + num);
>         this.notify();//唤醒其他正在等待的线程
>     }
>  
>     //生产资源
>     public synchronized void put() {
>         //如果资源满了，就进入阻塞状态
>         while (num == size) {//这里jdk源码里推荐用while，因为有可能出现虚假唤醒，所以要再次确认
>             try {
>                 System.out.println("生产者进入等待");
>                 this.wait();//线程进入阻塞状态，并释放锁
>  
>             } catch (InterruptedException e) {
>                 e.printStackTrace();
>             }
>         }
>  
>         num++;
>         System.out.println("生产者线程为:" + Thread.currentThread().getName() + "--资源数量:" + num);
>         this.notify();//唤醒其他正在等待的线程
>     }
> }
> ```
>
> ```
> // 消费者
> public class Consumer implements Runnable {
>  
>     private Resource resource;
>  
>     public Consumer(Resource resource) {
>         this.resource = resource;
>     }
>  
>     @Override
>     public void run() {
>         while (true){
>             resource.remove();
>         }
>  
>     }
> }
> ```
>
> ```
>  // 生产者
> public class Producer implements Runnable {
>  
>     private Resource resource;
>  
>     public Producer(Resource resource){
>         this.resource=resource;
>     }
>  
>     @Override
>     public void run() {
>         while (true){
>             resource.put();
>         }
>     }
> }
> ```
>
> ```
> // 测试代码
> public class TestConsumerAndProducer {
>  
>     public static void main(String[] args) {
>         Resource resource = new Resource();
>         //生产线程
>         Producer p1 = new Producer(resource);
>         //消费线程
>         Consumer c1 = new Consumer(resource);
>  
>         new Thread(p1).start();
>  
>         new Thread(c1).start();
>     }
> }
> ```
>
> **方法二：lock, condition, await, signal**
>
> ```
>  // 资源
> public class Resource {
>     //当前资源的数量
>     private int num = 0;
>     //当前资源的上限
>     private int size = 10;
>     private Lock lock = new ReentrantLock();//创建锁对象
>     private Condition condition = lock.newCondition();//创建锁的条件，情况
>  
>     //消费资源
>     public void remove() {
>         try {
>             lock.lock();//开启锁
>             //如果num为0，没有资源了，需要等待
>             while (num == 0) {//这里jdk源码里推荐用while，因为有可能出现虚假唤醒，所以要再次确认
>                 try {
>                     System.out.println("消费者进入等待");
>                     condition.await();//线程等待，并释放锁
>                 } catch (InterruptedException e) {
>                     e.printStackTrace();
>                 }
>             }
>             //如果线程可以执行到这里，说明资源里有资源可以消费
>             num--;
>             System.out.println("消费者线程为:" + Thread.currentThread().getName() + "--资源数量:" + num);
>             condition.signal();//唤醒其他等待的线程
>         } finally {
>             lock.unlock();//释放锁
>         }
>  
>     }
>  
>     //生产资源
>     public  void put() {
>         try {
>             lock.lock();//开启锁
>             //如果资源满了，就进入阻塞状态
>             while (num == size) {//这里jdk源码里推荐用while，因为有可能出现虚假唤醒，所以要再次确认
>                 try {
>                     System.out.println("生产者进入等待");
>                     condition.await();//线程等待，并释放锁
>                 } catch (InterruptedException e) {
>                     e.printStackTrace();
>                 }
>             }
>             num++;//如果线程执行到这里，说明资源未满，可以开始生产
>             System.out.println("生产者线程为:" + Thread.currentThread().getName() + "--资源数量:" + num);
>             condition.signal();//唤醒其他等待的线程
>         }finally {
>             lock.unlock();//释放锁
>         }
>  
>     }
> }
> ```
>
> ```
> // 消费者
> public class Consumer implements Runnable {
>  
>     private Resource resource;
>  
>     public Consumer(Resource resource) {
>         this.resource = resource;
>     }
>  
>     @Override
>     public void run() {
>         while (true){
>             resource.remove();
>         }
>  
>     }
> }
> ```
>
> ```
> // 生产者
> public class Producer implements Runnable {
>  
>     private Resource resource;
>  
>     public Producer(Resource resource){
>         this.resource=resource;
>     }
>  
>     @Override
>     public void run() {
>         while (true){
>             resource.put();
>         }
>     }
> }
> ```
>
> ```
> // 测试代码
> public class TestConsumerAndProducer {
>  
>     public static void main(String[] args) {
>         Resource resource = new Resource();
>         //生产线程
>         Producer p1 = new Producer(resource);
>         //消费线程
>         Consumer c1 = new Consumer(resource);
>  
>         new Thread(p1).start();
>  
>         new Thread(c1).start();
>     }
> }
> ```
>
> **结果**
>
> ![](https://img-blog.csdn.net/20180911135448646?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA0NTIzODg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### 53.用Java写代码来解决生产者——消费者问题。

#### 54.什么是竞争条件？你怎样发现和解决竞争？

> 当两个或以上的线程对同一个数据进行操作的时候，可能会产生“竞争条件”的现象。
>
> 解决：如果在一个线程对数据进行操作的时候，禁止另外一个线程操作此数据，那么，就能很好的解决以上的问题了。这种操作叫做给线程枷锁。

#### 55. 为什么我们调用start()方法时会执行run()方法，为什么我们不能直接调用run()方法？ ####
> 见题6；
> 
#### 56. Java中你怎样唤醒一个阻塞的线程？ ####

> 如果线程是因为调用了wait()、sleep()或者join()方法而导致的阻塞，可以中断线程，并且通过抛出InterruptedException来唤醒它；如果线程遇到了IO阻塞，无能为力，因为IO是操作系统实现的，Java代码并没有办法直接接触到操作系统。以下是详细的唤醒方法：
>
> 1. sleep() 方法
>
> sleep（毫秒），指定以毫秒为单位的时间，使线程在该时间内进入线程阻塞状态，期间得不到cpu的时间片，等到时间过去了，线程重新进入可执行状态。（暂停线程，不会释放锁）
>
> 2.suspend() 和 resume() 方法
>
> 挂起和唤醒线程，suspend e()使线程进入阻塞状态，只有对应的resume e()被调用的时候，线程才会进入可执行状态。（不建议用，容易发生死锁）
>
> 3. yield() 方法
>
> 会使的线程放弃当前分得的cpu时间片，但此时线程任然处于可执行状态，随时可以再次分得cpu时间片。yield()方法只能使同优先级的线程有执行的机会。调用 yield()的效果等价于调度程序认为该线程已执行了足够的时间从而转到另一个线程。（暂停当前正在执行的线程，并执行其他线程，且让出的时间不可知）
>
> 4.wait() 和 notify() 方法
>
> 两个方法搭配使用，wait()使线程进入阻塞状态，调用notify()时，线程进入可执行状态。wait()内可加或不加参数，加参数时是以毫秒为单位，当到了指定时间或调用notify()方法时，进入可执行状态。（属于Object类，而不属于Thread类，wait()会先释放锁住的对象，然后再执行等待的动作。由于wait()所等待的对象必须先锁住，因此，它只能用在同步化程序段或者同步化方法内，否则，会抛出异常IllegalMonitorStateException.）
>
> 5.join()方法
>
> 也叫线程加入。是当前线程A调用另一个线程B的join()方法，当前线程转A入阻塞状态，直到线程B运行结束，线程A才由阻塞状态转为可执行状态。
>
> 以上是Java线程唤醒和阻塞的五种常用方法，不同的方法有不同的特点，其中wait() 和 notify()是其中功能最强大、使用最灵活的方法，但这也导致了它们效率较低、较容易出错的特性，因此，在实际应用中应灵活运用各种方法，以达到期望的目的与效果！

#### 57. 什么是不可变对象，它对写并发应用有什么帮助？

> 1、 不可变对象(Immutable Objects)即对象一旦被创建它的状态（对象的数据，也即对象属性值）就不能改变，反之即为可变对象(Mutable Objects)。
>
> 2、 不可变对象的类即为不可变类(Immutable Class)。Java 平台类库中包含许多不可变类，如 String、基本类型的包装类、BigInteger 和 BigDecimal 等。
>
> 3、 只有满足如下状态，一个对象才是不可变的；
>
> - 它的状态不能在创建后再被修改；
>
>
> - 所有域都是 final 类型；并且，它被正确创建（创建期间没有发生 this 引用的逸出）。
>
> 不可变对象保证了对象的内存可见性，对不可变对象的读取不需要进行额外的同步手段，提升了代码执行效率。
>
> final修饰一个类时，表明这个类时不可变类，也不能被继承，final类中所有的成员方法都会被隐式的指定为final方法，final方法不可被重写；当final修饰一个基本数据类型时，表示该基本数据类型的值一旦被初始化便不能发生变化。如果final修饰一个引用类型时，则对其初始化后便不能再指向其他对象了，但该引用指向的对象内容可以发生变化。本质上final要求值/引用地址值不能发生变化。

#### 58. 你在多线程环境中遇到的共同的问题是什么？你是怎么解决它的？

>  多线程和并发程序中常遇到的有Memory-interface、竞争条件、死锁、活锁和饥饿。问题是没有止境的，如果你弄错了，将很难发现和调试。 

#### 59. Java 中能创建 volatile数组吗

> volatile是可以修饰数组的，所以java中可以创建volatile数组；例如 volatile int array[10]是指array的地址是volatile的，而不是数组元素的值是volatile的。

#### 61. volatile 能使得一个非原子操作变成原子操作吗

>  能，一个典型的例子是在类中有一个 long 类型的成员变量。如果你知道该成员变量会被多个线程访问，如计数器、价格等，你最好是将其设置为 volatile。为什么?因为 Java 中读取 long 类型变量不是原子的，需要分成两步，如果一个线程正在修改该 long 变量的值，另一个线程可能只能看到该值的一半(前 32 位)。但是对一个 volatile 型的 long 或 double 变量的读写是原子。 

#### 62. 你是如何调用 wait（）方法的？使用 if 块还是循环？为什么？

#### 63. 我们能创建一个包含可变对象的不可变对象吗？

> 是的，我们是可以创建一个包含可变对象的不可变对象的，你只需要谨慎一点，不要共享可变对象的引用就可以了，如果需要变化时，就返回原对象的一个拷贝。最常见的例子就是对象中包含一个日期对象的引用

64. 在多线程环境下，SimpleDateFormat是线程安全的吗

65. 为什么Java中 wait 方法需要在 synchronized 的方法中调用？

66. BlockingQueue，CountDownLatch及Semeaphore的使用场景

67. Java中interrupted 和 isInterruptedd方法的区别？

68. 怎么检测一个线程是否持有对象监视器

    #### 69. 什么情况会导致线程阻塞
>  在某一时刻某一个线程在运行一段代码的时候，这时候另一个线程也需要运行，但是在运行过程中的那个线程执行完成之前，另一个线程是无法获取到CPU执行权的（调用sleep方法是进入到睡眠暂停状态，但是CPU执行权并没有交出去，而调用wait方法则是将CPU执行权交给另一个线程），这个时候就会造成线程阻塞。 
>
> 1.睡眠状态：当一个线程执行代码的时候调用了sleep方法后，线程处于睡眠状态，需要设置一个睡眠时间，此时有其他线程需要执行时就会造成线程阻塞，而且sleep方法被调用之后，线程不会释放锁对象，也就是说锁还在该线程手里，CPU执行权还在自己手里，等睡眠时间一过，该线程就会进入就绪状态，典型的“占着茅坑不拉屎”；
>
> 2.等待状态：当一个线程正在运行时，调用了wait方法，此时该线程需要交出CPU执行权，也就是将锁释放出去，交给另一个线程，该线程进入等待状态，但与睡眠状态不一样的是，进入等待状态的线程不需要设置睡眠时间，但是需要执行notify方法或者notifyall方法来对其唤醒，自己是不会主动醒来的，等被唤醒之后，该线程也会进入就绪状态，但是进入就绪状态的该线程手里是没有执行权的，也就是没有锁，而睡眠状态的线程一旦苏醒，进入就绪状态时是自己还拿着锁的。等待状态的线程苏醒后，就是典型的“物是人非，大权旁落“；
>
> 3.礼让状态：当一个线程正在运行时，调用了yield方法之后，该线程会将执行权礼让给同等级的线程或者比它高一级的线程优先执行，此时该线程有可能只执行了一部分而此时把执行权礼让给了其他线程，这个时候也会进入阻塞状态，但是该线程会随时可能又被分配到执行权，这就很”中国化的线程“了，比较讲究谦让；
>
> 4.自闭状态：当一个线程正在运行时，调用了一个join方法，此时该线程会进入阻塞状态，另一个线程会运行，直到运行结束后，原线程才会进入就绪状态。这个比较像是”走后门“，本来该先把你的事情解决完了再解决后边的人的事情，但是这时候有走后门的人，那就会停止给你解决，而优先把走后门的人事情解决了；
>
> 5.suspend() 和 resume() ：这两个方法是配套使用的，suspend() 是让线程进入阻塞状态，它的解药就是resume()，没有resume()它自己是不会恢复的，由于这种比较容易出现死锁现象，所以jdk1.5之后就已经被废除了，这对就是相爱相杀的一对。

#### 64.如何在两个线程间共享数据

> 

#### 64.Thread.sleep(1000)的作用是什么？

> 线程睡眠1秒钟

#### 65.使用多线程可能带来什么问题

- 线程安全问题

- 内存泄漏
- 上下文切换导致性能消耗
- 死锁

#### 66.说说线程的生命周期和状态?

1. 什么是上下文切换
2. Java Monitor 的工作机理
3. 按线程池内部机制，当提交新任务时，有哪些异常要考虑。
4. 线程池都有哪几种工作队列？
5. 说说几种常见的线程池及使用场景?
6. 使用无界队列的线程池会导致内存飙升吗？
#### 80. 为什么阿里发布的 Java开发手册中强制线程池不允许使用 Executors 去创建？ ####
> 阿里巴巴的java手册里面说到，线程池的创建不要用JDK提供的那些简单方法，容易埋坑，要用ThreadPoolExecutor构造函数，来明确各个参数的意义，这样可以避免出错，代码可读性也很好。
81. Future有缺陷嘛？
