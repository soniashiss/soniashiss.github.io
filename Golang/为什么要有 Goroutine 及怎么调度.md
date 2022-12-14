# Goroutine及其调度学习总结


**Created in 2019，updated in 2022**

## 1.是什么

### 1.1 Goroutine的定义

[A Tour of Go](https://tour.golang.org/concurrency/1)给出的官方定义是：

> A goroutine is a lightweight thread managed by the Go runtime.

划重点: **lightweight thread**, **managed by the Go runtime**

所以最简单的理解：

1. goroutine是一个线程：从概念上理解，可以认为goroutine本身是一个线程。即从使用上来说，具有线程的特性。

2. goroutine比一般意义上的线程还要轻一些。

3. goroutine是由Go runtime来管理的

到这里其实已经有几个疑问了：

1. goroutine比一般的线程轻在哪里，又有什么区别？

2. 为什么Go要自己做一个goroutine而不是直接用thread？

3. 怎么实现的？

先看看goroutine和线程有什么区别：

### 1.2 线程的定义

线程是一个操作系统的概念，要说线程，必定离不开进程。进程的官方概念比较抽象，还有狭义和广义之分，这里不展开。由于后续我们会说gorouinte比线程更“轻”，一般这个“轻”指的是内存占用小、结构简单（因而处理更方便），所以这里我们就从进程和线程的内存布局说起。

#### 1.2.1 进程的内存布局

Linux 0.0.1版本的进程（此时还没有线程）的内存布局：（引用自[simpleLinux 进程内存分布](https://github.com/1184893257/simplelinux/blob/master/mem.md)）

![Linux进程内存分布](https://camo.githubusercontent.com/9ed1cb55f6baacfe9d64580b4b65f1bca45f7dca/687474703a2f2f666d6e2e7272696d672e636f6d2f666d6e3036312f32303132313230362f313932352f6f726967696e616c5f745879675f363163383030303030353962313138642e6a7067)

从这里可以看到，进程是一个很重的数据结构，当进程切换时，需要保存当前进程执行到哪里、进程跑到当前状态时CPU的各个寄存器里的内容是什么等等等称之为上下文的东西，然后再把要执行进程的上下文加载一遍。（此处仅简单描述，不严谨）

我们知道，现代CPU的速度远快于RAM等的速度，而CPU执行任务（执行进程某个代码片段）前，是要求用到的所有资源都准备好的（比如寄存器、比如RAM、比如网络等等），因此速度不匹配必然导致执行过程中CPU需要等。

同时，由于某些任务可能不止干一件事情（比如Word写文档的时候，可能你在输入而Word在给你检查语法），所以这个时候我们可以将两件事情穿插进行，准备好所有资源的事情就可以跑，跑到需要等待资源的时候，就去干另一件事情。这里的“事情”,就可以认为是线程（继续不严谨）。同时，由于这两件事情都是在一个任务下的，关联度很高，所以大部分情况下它们的资源都是可以共用的。由此产生了线程的内存布局：

#### 1.2.2 线程的内存布局

![Linux线程内存分布](https://camo.githubusercontent.com/7cedb090a73d847ad5d572bb735cd53a462af613/687474703a2f2f666d6e2e786e7069632e636f6d2f666d6e3035362f32303132313230362f313932352f6f726967696e616c5f323262635f323535363030303030356138313138632e6a7067)

其实说Linux的线程内存分布并不准确，因为**线程的容器还是进程**。这其实是加入了线程概念后，现代操作系统（主要指Linux）的进程内存布局。

从图上来看，同一个进程（地址空间内）,不同线程之间除了栈不共享，其余资源都是可以共享访问的。于是就达到了上面说的目的：等待资源时，CPU可以切换到另一个线程执行，于是CPU就可以不空闲地“做事”，而由于线程间大部分资源共享，所以切换线程时，并不需要做上面说的一整套上下文切换的操作，于是降低了“做事”切换的成本。

#### 1.3 Goroutine和线程的区别

从上面对线程的介绍来看，线程其实已经能满足提高效率让CPU少空转的需求了，但是为什么还要用Goroutine？那就先从Goroutine和线程的区别来看：

1) **内存占用**：一般情况下，linux线程(main除外，下同)堆栈默认分配占用空间为8MB（最大可用8MB）; 而goroutine(G0除外，同下)默认分配为2KB
2) **栈增长**：linux线程超出8MB需要修改参数来增大，非动态扩展；goroutine超出时，runtime动态扩展
3) **数目限制**：linux一个进程可创建最大线程数65530，用户创建最大线程数默认1024，可通过参数修改（注意重启）；golang无限制（使用不当等等不算）
4) **内核态和用户态**：goroutine为用户态，由runtime调度；linux的线程可以有内核态也可以有用户态，大多数情况下由内核调度

## 2.为什么

既然Linux已经有了进程/线程模型，理论来说golang直接适用系统的模型就可以，还可以省掉管理、调度的逻辑，简化整个系统，那为什么golang还要自己做Goroutine？

此[链接](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)说得非常好（有部分个人理解）：

1) **复杂** 若直接通过语言来创建进程/线程，由操作系统调度，则需要考虑创建参数、多个线程之间的通信、stack等等一系列内容，并且将这些全部设置合理是需要一定经验的。

2) **切换代价大** 虽然线程切换代价比进程小，但是线程切换时，CPU仍然需要做一系列上下文切换（比如保存暂停点、寄存器内容然后把另外一个的读进来）。而如果自己做一个调度器（怎么实现的先不管），那么就可以让CPU不要频繁进行线程的切换操作（而由线程内部来调度不同的执行单元），从而降低消耗。(实际上，goroutine切换时，只需要切换PC、SP、DX三个寄存器)

3) **内存占用小** 区别中已经提到过，thread默认情况下8MB，而goroutine 2KB

## 3.如何实现的

### 3.1 数据结构

golang的实现过程中有三个重要数据结构，M,P和G。（最早版本的M和G在golang的runtime.h中定义，P在proc.h定义；较新版本（不完全确定版本，但可以肯定1.5以后一定）在https://golang.org/src/runtime/runtime2.go 中定义）

M: Machine，真正执行代码的worker thread。其结构字段众多，会维护mcache等内容，另外比较重要的是会存储当前正在运行的G、以及关联的P。(PS:不好理解的话，可以认为是P执行的容器)

P: Processor，逻辑线程， 可以认为是在M上的调度器。其维护了一个所有需要它执行的goroutine队列。（是不是感觉和M重复？往下看）

G: Goroutine, goroutine的执行上下文。如goroutin的栈、instruction pointer和正在等待的channel等等

### 3.2 如何调度

#### 3.2.1 早期版本：G-M模型

Go 1.0时，gorutine的调度为G-M模型，即Machine直接关联管理G。具体调度方式不展开讲，只需知道此模型限制了Go并发程序的伸缩性，尤其对于高吞吐或者并行计算需求的程序而言。

具体可见[Scalabe Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#!)，此处做个简单摘要：

* 单一全局互斥锁和集中状态存储导致goroutine的所有操作（如创建、重新调度等）都要上锁；（限制并发性能）

* M的问题：

    * goroutine传递问题：M和M之间经常传递“可运行”的goroutine，导致额外的调度开销。（即性能损耗）

    * 内存问题：每个M都做内存缓存，因此内存占用高，数据的局部性较差。

* syscall调用形成的worker thread的阻塞和解除，造成额外的性能开销

因此有了后来的G-P-M模型，也解释了3.1中M和P“重复”的问题

#### 3.2.2 G-P-M模型

整体来说，G-P-M模型下，golang调度的整体模型如下：（引用自[也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)）

![Goroutine调度原理图](https://tonybai.com/wp-content/uploads/goroutine-scheduler-model.png)

整体来说，调度过程是：每个M都要绑定一个P，每个P都有一个goroutine的queue（局部的，Local Queue），goroutine执行时，从P的queue选出一个goroutine（G）由M执行。

当然，调度过程需要处理各种不同的情况：

1. M跑着跑着，syscall进入interrupt了怎么办？

    P会转投另外一个M。这个另外一个M，可以是“空着”的M，也可以是新创建的M（此时M必然是“空着”的）

2. M interrupt返回了怎么办？

    当M返回时，它本来的P已经跑了，这时M会试图“偷”一个P过来。这个“偷”，当然是指那些没有绑定M的P，或者是M阻塞的P。如果成功，那么就可以进入上面说的调度过程；如果整个系统里都没有这种P了，那么M就先把自己手上这个刚interrupt回来的goroutine(G)放进global queue里，然后自己睡觉去了。

3. Global Queue里的G怎么办？

    系统里的P会定期检查Global Queue，把里面的G放到自己的Local Queue里，然后就可以愉快地跑任务了。

4. P的Local Queue里没有G怎么办？

    先检查Global Queue，如果也没有G，那么就和M一样，从别的P那边“偷”一些G回来继续跑。

总结一下，就是下面这张图：

![goroutine调度](https://colobu.com/2017/05/04/go-scheduler/go-sched.png)

### 3.2.3 抢占式调度
上述GMP模型有一个缺点：如果某个goroutine死循环了，那么那个M就一直在跑死循环的goroutine了，没有办法被别的goroutine使用。
为此，Go加入了抢占式调度，主要在两个时间地方可以检测：
1）某段function 进入之前，Go的编译器会自动插入一小段代码检测是否需要抢占（tbc，什么版本，为什么，什么原理，适用场景）
2）内存回收时（tbc）

# 4.总结

* Goroutine是Golang Runtime管理的、轻量级的线程，是Golang程序调度执行的最小单元。
* Golang使用Runtinme和Goroutine是为了降低CPU切换线程开销，提升并发性能；同时带来的一个好处是降低使用编程语言来进行并发编程时对于多进程/线程/操作系统等参数的设置经验要求（当然，也简化了编程操作）。
* 当前主流版本Golang均采用M-P-G调度模型，其中M为真正执行CPU指令的worker thread，P为逻辑上的Processor，G为Goroutine执行上下文。

#CS/Go 