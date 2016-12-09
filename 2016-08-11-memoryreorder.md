---
title: 深入探索内存并发系列(五)-将内存乱序逮个正着
date: 2016-08-11 11:03:27
tags:
categories: High-performance
---

当用C/C++编写无锁代码时，一定要小心谨慎，以保证正确的内存顺序。不然的话，会发生一些诡异的事情。


<!-- more -->

Intel在[x86/x64体系结构手册](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)的Volume 3, §8.2.3 中列出了一些可能会发生的诡异的事情。这里介绍其中一个最简单的例子。假设在内存中有两个整型变量`x`和`y`，都初始化为0。两个处理器并行执行下面的机器码：

![](http://7xppf1.com1.z0.glb.clouddn.com/memoryreorder1.png)

不要被上面的汇编代码给吓坏了。这个例子的确是阐述CPU执行顺序的最好方式。每个处理器将1写入其中一个整型变量中，然后将另一个整型变量读取到寄存器中。（`r1`和`r2`只是x86中真实寄存器-如eax寄存器-的代表符号).

现在不管哪个处理器先将1写入内存，都想当然的认为另一个处理器会读到这个值，这就意味着最后结果中要么`r1=1`，要么`r2=1`，要么这两个结果同时满足。但根据Intel手册，却不是这么回事。手册上说在这个例子里，最终`r1`和`r2`的值都有可能等于0。至少可以这么说，这个结果是不太符合大家直觉的。

可以这么理解：Intel x86/x64处理器，和大部分处理器家族一样，在保证不改变一个单线程程序执行的基础上，会根据一定的规则将机器指令对内存的操作顺序重新排序。具体来说，对于不同内存变量的写读操作，处理器保留乱序的权利<sup>注1</sup>。 结果就好像是指令就是按照下图这个顺序执行的：

![](http://7xppf1.com1.z0.glb.clouddn.com/memoryorder2.png)


### 指令乱序重现


能被告知这种诡异的事情会发生总是好的，但眼见才为实。这也就是我为什么要写个小程序来说明这种重新排序会发生的原因。你可以在[这里](http://7xppf1.com1.z0.glb.clouddn.com/ordering.zip)下载源码。

代码样例分别包含Win32和POSIX版本。代码中会派生出两个工作线程不断重复上述的事务，主线程用来同步这些工作并检查最终结果。

下面是第一个工作线程的源码。`X`,`Y`,`r1`和`r2`都是全局变量，POSIX信号量用来协调每个循环的开始和结束。

```C++
sem_t beginSema1;
sem_t endSema;

int X, Y;
int r1, r2;

void *thread1Func(void *param)
{
    MersenneTwister random(1);                // Initialize random number generator
    for (;;)                                  // Loop indefinitely
    {
        sem_wait(&beginSema1);                // Wait for signal from main thread
        while (random.integer() % 8 != 0) {}  // Add a short, random delay

        // ----- THE TRANSACTION! -----
        X = 1;
        asm volatile("" ::: "memory");        // Prevent compiler reordering
        r1 = Y;

        sem_post(&endSema);                   // Notify transaction complete
    }
    return NULL;  // Never returns
};
```

每个事务前用一个短暂、随机的延迟用来错开线程的时间。记住，这里有两个工作线程，我们要试着将他们的指令重叠。随机延迟是用我前面文章，[锁不慢；锁竞争慢
](http://www.chongh.wiki/blog/2016/06/20/mutexperf/)和[实现递归锁](http://www.chongh.wiki/blog/2016/07/11/recursivemutex/)的使用过的`MersenneTwister`来实现的。


别被上面代码中的`asm volatile`给吓坏了。其作用就是直接告诉[GCC编译器在生成机器码的时候不要重新安排store和load操作](https://en.wikipedia.org/wiki/Memory_ordering#Compiler_memory_barrier)，以防在优化期间做了手脚<sup>注2</sup>. 我们可以检查下面的汇编代码来验证这个过程。意料之中，store和load操作按照我们想要的顺序执行。之后的指令将`eax`寄存器中的结果写回到全局变量`r1`中。

```C++
$ gcc -O2 -c -S -masm=intel ordering.cpp
$ cat ordering.s
    ...
    mov    DWORD PTR _X, 1
    mov    eax, DWORD PTR _Y
    mov    DWORD PTR _r1, eax
    ...
```

主线程的源码如下。其执行所有的管理工作。初始化后，进入无限循环，在每次迭代开始工作线程之前会重新设置X和Y为0。

注意`sem_post`之前所有有可能发生的共享内存写操作，以及`sem_wait`之后所有有可能发生的共享内存读操作。工作线程在和主线程通信的过程中也要遵守同样的规则。信号量为每个平台提供了acquire和release语义。这意味着我们可以保证初始值`X=0`和`Y=0`可以完全传播到工作线程中，`r1`和`r2`的结果也会被完整传回来。换句话说，信号量阻止了乱序<sup>注3</sup>，可以让我们全心关注实验本身。

```C++
int main()
{
    // Initialize the semaphores
    sem_init(&beginSema1, 0, 0);
    sem_init(&beginSema2, 0, 0);
    sem_init(&endSema, 0, 0);

    // Spawn the threads
    pthread_t thread1, thread2;
    pthread_create(&thread1, NULL, thread1Func, NULL);
    pthread_create(&thread2, NULL, thread2Func, NULL);

    // Repeat the experiment ad infinitum
    int detected = 0;
    for (int iterations = 1; ; iterations++)
    {
        // Reset X and Y
        X = 0;
        Y = 0;
        // Signal both threads
        sem_post(&beginSema1);
        sem_post(&beginSema2);
        // Wait for both threads
        sem_wait(&endSema);
        sem_wait(&endSema);
        // Check if there was a simultaneous reorder
        if (r1 == 0 && r2 == 0)
        {
            detected++;
            printf("%d reorders detected after %d iterations\n", detected, iterations);
        }
    }
    return 0;  // Never returns
}
```

最后，关键时刻到了。这是在Intel Xeon W3520中运行[Cygin](http://www.cygwin.com/)的输出。

![](http://7xppf1.com1.z0.glb.clouddn.com/memoryorder3.png)


在运行期间，每6600次迭代差不多能检测到一次乱序。当我在Core 2 Duo E6300处理器Ubuntu系统中测试时，乱序的次数更少见。大家开始对这种微妙的时间bug是如何能蔓延到无锁代码中而不被检测到感到刺激。

现在，假设你想避免这种乱序，至少有两种方法可以做到。其中一种方法就是设置线程亲和力（thread affinities），以让两个工作线程能在同一个CPU核上独立运行。Pthreads中没有可移植的方法设置亲和力，但在Linux上，可以这样来实现：

```C++
  cpu_set_t cpus;
  CPU_ZERO(&cpus);
  CPU_SET(0, &cpus);
  pthread_setaffinity_np(thread1, sizeof(cpu_set_t), &cpus);
  pthread_setaffinity_np(thread2, sizeof(cpu_set_t), &cpus);
```

这样修改之后，乱序就不会发生了。那是因为尽管当线程在任一时间抢占处理器并被重新调度，单个处理器绝不会让自己的操作乱序<sup>注4</sup>。当然了，将两个线程都锁到一个单独的核中，其它核就用不上了。

与此相关的是，我在Playstation 3上编译并运行了这份代码，没有检测到乱序的情况。这意味着（不能确信)在PPU里的两个[硬件线程](https://en.wikipedia.org/wiki/Cell_(microprocessor)#Power_Processor_Element_.28PPE.29)可能会充当一个单处理器，具有细粒度的硬件调度能力。

### 用storeload barrier来避免


在这个例子中，另一种阻止内存乱序的方法是在两条指令间引入一个CPU级的Memory Barrier。在这里，我们要避免store操作紧接load操作的乱序情况。用惯用的barrier行话来说， 我们需要的是一个Storeload barrier。

在x86/x64处理器中，没有特定的指令用来充当Storeload barrier,但有其它的一些指令能做到甚至更多的事情。`mfence`指令就是一个full memory barrier，可以避免任何形式的内存乱序。在GCC中，实现方式如下：

```C++
    for (;;)                                  // Loop indefinitely
    {
        sem_wait(&beginSema1);                // Wait for signal from main thread
        while (random.integer() % 8 != 0) {}  // Add a short, random delay

        // ----- THE TRANSACTION! -----
        X = 1;
        asm volatile("mfence" ::: "memory");  // Prevent memory reordering
        r1 = Y;

        sem_post(&endSema);                   // Notify transaction complete
    }
```

同样地，可以检查下面的汇编代码来验证。

```C++
 ...
    mov    DWORD PTR _X, 1
    mfence
    mov    eax, DWORD PTR _Y
    mov    DWORD PTR _r1, eax
    ...
```

这样修改之后，内存乱序就不会发生了，并且我们依然允许两个线程分别运行在不同的CPU核中<sup>注5</sup>。

### 类似指令与不同平台

有趣的是，`mfence`并不是x86/x64平台中能唯一充当full memory barrier的指令。在这些处理器中，假设你不使用SSE指令或者写结合内存(Write-combined Memory)（例子中也并没有用到），任何带lock的指令，比如`xchg`，也能作为一个full memory barrier。实际上，Microsoft C++编译器在你使用`MemoryBarrier`时会生成`xchg`指令，至少Visualstudio 2008是这么做的

`mfence`指令是x86/x64平台独有的<sup>注6</sup>。如果你想让代码具有可移植性，可以将这种固有特性写成一个预处理的宏。Linux内核将其封装成一个叫做`smp_mb`的宏，以及相关的宏`smp_rmb`和`smp_wmb`宏<sup>注7</sup>，并提供了[在不同架构中的不同实现方法](http://lxr.free-electrons.com/ident?i=smp_mb)。 例如，在PowerPC中，`smp_mb`宏是通过`sync`来实现的.

在这些不同的CPU家族中，每种CPU都有各自的指令来保证内存访问顺序，每个编译器通过不同的内置属性展现出来，每种跨平台的项目都会实现自己的可移植层。 然而，这些都不能让无锁编程变得更加简单。 这就是[C++11原子库标准](http://www.open-std.org/JTC1/sc22/wg21/docs/papers/2007/n2427.html)在最近被提出来的部分原因。这是标准化的一次尝试，可能会让写可移植性的无锁代码变得更加简单。

注释

注1：注意，这里说的是写读乱序，而且是对不同变量的写读操作的乱序。在Intel x86/x64处理器中，读读、写写、读写、以及写读同一个内存变量，CPU是不会乱序的。

注2：`asm volatile("" ::: "memory")`是一条编译器级别的Memory Barrier，可以防止编译器对相邻指令进行乱序，但是它对CPU乱序是没有影响的；也就是说它仅仅束缚了编译器的乱序优化，不会阻止CPU可能的乱序执行。这么做自然是将编译器的干扰和影响降到最低，好让我们专注观察CPU的执行行为。

注3：请务必注意，这里说的阻止乱序是指防止了向`sem_wait`和`sem_post`之外的乱序，不阻止它们之间的乱序。举个例子：

```c++
mutex.lock();
a=1;
b=2;
mutex.unlock;
```

这里lock保证了`a=1`和`b=1`这两行代码不会被拉到lock之上执行；同理，也不会被拉到unlock之下执行。

因此，我们lock和unlock分别提供了acquire语义和release语义。但是lock和unlock之间的代码是允许乱序的，也可能发生乱序的，而这正是这个实验的目的。

这里，lock对应文中的`sem_wait`，unlock对应`sem_post`。借此机会，读者可以对锁有更好的认识。

注4：也就是说，单核多线程、多核单线程程序不用担心memory reordering问题，只有多核多线程才需要小心谨慎。为什么呢？请看下面的注4。

注5：到目前为止，读者可能会对通篇文章里的内容有两个疑问：

1，为什么CPU要乱序执行，难道是考虑性能吗？那为什么乱序就能提升性能？

2，为什么在Intel X86/64架构下，就只有写读（Store Load）发生乱序呢？读读呢？读写呢？

要明白这两个问题，我们首先得知道cache coherency，也就是所谓的cache一致性。

在现代计算机里，一般包含至少三种角色：cpu、cache、内存。一般说来，内存只有一个；CPU Core有多个；cache有多级，cache的基本块单位是cacheline，大小一般是64B-256B。

每个cpu core有自己的私有的cache(有一级cache是共享的)，而cache只是内存的副本。那么这就带来一个问题：如何保证每个cpu core中的cache是一致的？

在广泛使用的cache一致性协议即MESI协议中，cacheline有四种状态：Modified、Exclusive、Shared、Invalid，分别表示修改、独占、共享、无效。

当某个cpu core写一个内存变量时，往往是（先）只修改cache，那么这就会导致不一致。为了保证一致，需要先把其他core的对应的cacheline都invalid掉，给其他core们发送invalid消息，然后等待它们的response。

这个过程是耗时的，需要执行写变量的core等待，阻塞了它后面的操作。为了解决这个问题，cpu core往往有自己专属的store buffer。

等待其他core给它response的时候，就可以先写store buffer，然后继续后面的读操作，对外表现就是写读乱序。

因为写操作是写到store buffer中的，而store buffer是私有的，对其他core是透明的，core1无法访问core2的store buffer。因此其他core读不到这样的修改。

这就是大概的原理。MESI协议非常复杂，背后的技术也很有意思。

注6：不建议使用这么原生(raw) 的memory barrier。在GCC下，推荐使用`__sync_synchronize`。

注7：X86下，`smp_wmb`是一个空宏，什么也不做；而`smp_rmb`则不是。想想看，为什么。

注8：作为练习，请读者朋友们分析以下问题，其中A、B、C的初值都是0

```c++
thread1(void)
{
  A = 1;
  cpu_barrier();
  B = 1;
}
thread2(void)
{
  while (B != 1)
    continue;
  compiler_barrier();
  C = 1;
}
thread3(void)
{
  while (C != 1)
    continue;
  compiler_barrier();
  assert(A != 0);
}
```

其中，`cpu_barrier`是cpu级别的memory barrier，影响cpu和编译器，防止它们乱序；`compiler_barrier`只防止编译器乱序。

问题：thread3中的断言是否可能会失败？为什么？别急着回答，考虑平台是否是x86？考虑单核多线程、多核单线程、多核多线程？

另外，这篇流传很广的文章有错，务必小心：http://blog.csdn.net/jnu_simba/article/details/22985913

注9：注意，我们的讨论只针对普通指令，对于SSE等特殊指令，情况可能完全不同。这点读者务必注意。



### Acknowledgement

本文由 [Diting0x](http://weibo.com/2767520802/profile?topnav=1&wvr=6&is_all=1) 与 [睡眼惺忪的小叶先森](http://weibo.com/yebangyu) 共同完成，在原文的基础上添加了许多精华注释，帮助大家理解。

感谢好友[小伙伴-小伙伴儿 ](http://weibo.com/u/1707881862?topnav=1&wvr=6&topsug=1&is_all=1)与[skyline09_ ](http://weibo.com/u/2495479763?topnav=1&wvr=6&topsug=1&is_all=1)阅读了初稿，并给出宝贵的意见。

原文： http://preshing.com/20120515/memory-reordering-caught-in-the-act/


本文遵守Attribution-NonCommercial-NoDerivatives 4.0 International License (CC BY-NC-ND 4.0)
仅为学习使用，未经博主同意，请勿转载
本系列文章已经获得了原作者preshing的授权。版权归原作者和本网站共同所有








