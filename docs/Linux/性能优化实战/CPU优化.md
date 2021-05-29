## CPU基础知识

### 进程与线程

**进程**（英语：process），是指计算机中已运行的程序。进程曾经是分时系统的基本运作单位。在面向进程设计的系统（如早期的UNIX，Linux 2.4及更早的版本）中，进程是程序的基本执行实体；在面向线程设计的系统（如当代多数操作系统、Linux 2.6及更新的版本）中，进程本身不是基本运行单位，而是线程的容器。

**线程**（英语：thread）是操作系统能够进行运算调度的最小单位。大部分情况下，它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。在Unix System V及SunOS中也被称为轻量进程（lightweight processes），但轻量进程更多指内核线程（kernel thread），而把用户线程（user thread）称为线程。

### CPU 调度

***

### 中断系统

是指处理器接收到来自硬件或软件的信号，提示发生了某个事件，应该被注意，这种情况就称为中断。通常，在接收到来自外围硬件（相对于中央处理器和内存）的异步信号，或来自软件的同步信号之后，处理器将会进行相应的硬件／软件处理。发出这样的信号称为进行中断请求（interrupt request，IRQ）。硬件中断导致处理器通过一个运行信息切换（context switch）来保存执行状态（以程序计数器和程序状态字等寄存器信息为主）；软件中断则通常作为CPU指令集中的一个指令，以可编程的方式直接指示这种运行信息切换，并将处理导向一段中断处理代码。

### CPU 缓存

是用于减少处理器访问内存所需平均时间的部件。在金字塔式存储体系中它位于自顶向下的第二层，仅次于CPU寄存器。其容量远小于内存，但速度却可以接近处理器的频率。当处理器发出内存访问请求时，会先查看缓存内是否有请求数据。如果存在（命中），则不经访问内存直接返回该数据；如果不存在（失效），则要先把内存中的相应数据载入缓存，再将其返回处理器。

### NUMA

非统一内存访问架构是一种为多处理器的电脑设计的内存架构，内存访问时间取决于内存相对于处理器的位置。在NUMA下，处理器访问它自己的本地内存的速度比非本地内存（内存位于另一个处理器，或者是处理器之间共享的内存）快一些。

非统一内存访问架构的特点是：被共享的内存物理上是分布式的，所有这些内存的集合就是全局地址空间。所以处理器访问这些内存的时间是不一样的，显然访问本地内存的速度要比访问全局共享内存或远程访问外地内存要快些。另外，NUMA中内存可能是分层的：本地内存，群内共享内存，全局共享内存。

## 性能指标

### 平均负载

```shell
# uptime 
 09:57:53 up 26 min,  1 user,  load average: 0.10, 0.13, 0.21
```

我们在登录系统初步检查CPU使用状态时基本上会用`uptime`或者`w`命令，这些命令的第一行显示内容的意思为：

* 当前系统时间
* 系统运行时间
* 正在登录的用户数
* 系统的一分钟、五分钟、十五分钟的平均负载

那什么是平均负载？

平均负载是指单位时间内，系统处于**可运行状态**和**不可中断状态**的平均进程数，也就是平均活跃进程数，它和 CPU 使用率并没有直接关系。

* 所谓可运行状态的进程，是指正在使用 CPU 或者正在等待 CPU 的进程，也就是我们常用 ps 命令看到的，处于 R 状态（Running 或 Runnable）的进程。

* 不可中断状态的进程则是正处于内核态关键流程中的进程，并且这些流程是不可打断的，比如最常见的是等待硬件设备的 I/O 响应，也就是我们在 ps 命令中看到的 D 状态（Uninterruptible Sleep，也称为 Disk Sleep）的进程。

因此，你可以简单理解为，平均负载其实就是平均活跃进程数。平均活跃进程数，直观上的理解就是单位时间内的活跃进程数，但它实际上是活跃进程数的指数衰减平均值。

**平均负载与CPU使用率**

前面已经说了平均负载是指单位时间内，处于可运行状态和不可中断状态的进程数。所以，它不仅包括了**正在使用 CPU** 的进程，还包括**等待 CPU** 和**等待 I/O** 的进程。

而 CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载并不一定完全对应。比如：

* CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
* I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
* 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。

**案例分析**

机器配置：2C、8G内存

> CPU 密集型进程

模拟一个CPU使用率100%的场景

```shell
# stress --cpu 1 --timeout 600
```

输出监控所有CPU数据：

```shell
# mpstat -P  ALL 5
07:19:57 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:20:02 PM  all   50.30    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   49.50
07:20:02 PM    0    0.40    0.00    0.40    0.00    0.00    0.00    0.00    0.00    0.00   99.20
07:20:02 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```

查看哪个进程导致CPU上升：

```shell
# pidstat -u 5 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/04/2021      _x86_64_        (2 CPU)

07:19:35 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:19:40 PM     0         9    0.00    0.20    0.00    0.20     0  rcu_sched
07:19:40 PM     0      8846    0.20    0.00    0.00    0.20     0  vmtoolsd
07:19:40 PM     0      9787  100.00    0.00    0.00  100.00     1  stress
07:19:40 PM     0      9926    0.00    0.20    0.00    0.20     0  pidstat

Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:        0         9    0.00    0.20    0.00    0.20     -  rcu_sched
Average:        0      8846    0.20    0.00    0.00    0.20     -  vmtoolsd
Average:        0      9787  100.00    0.00    0.00  100.00     -  stress
Average:        0      9926    0.00    0.20    0.00    0.20     -  pidstat
```

### CPU上下文切换

**进程上下文切换**

先把前一个任务的 CPU 上下文（也就是 CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。 而这些保存下来的上下文，会存储在系统内核中，并在任务重新调度执行时再次加载进来。这样就能保证任务原来的状态不受影响，让任务看起来还是连续运行。 

##### 系统调用

一次系统调用的过程，其实是发生了两次 CPU 上下文切换。（用户态-内核态-用户态）

* 进程是由内核来管理和调度的，进程的切换只能发生在内核态。所以，进程的上下文不仅包括了虚拟内存、栈、全局变量等用户空间的资源，还包括了内核堆栈、寄存器等内核空间的状态。

* 进程的上下文切换就比系统调用时多了一步：在保存内核态资源（当前进程的内核状态和 CPU 寄存器）之前，需要先把该进程的用户态资源（虚拟内存、栈等）保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。

##### 发生进程上下文切换的场景

1. 为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，就会被系统挂起，切换到其它正在等待 CPU 的进程运行。
2. 进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。
3. 当进程通过睡眠函数 sleep 这样的方法将自己主动挂起时，自然也会重新调度。
4. 当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行
5. 发生硬件中断时，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序。

**线程上下文切换**

线程与进程最大的区别在于：**线程是调度的基本单位，而进程则是资源拥有的基本单位**。说白了，所谓内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。

所以，对于线程和进程，我们可以这么理解：

- 当进程只有一个线程时，可以认为进程就等于线程。
- 当进程拥有多个线程时，这些线程会共享相同的虚拟内存和全局变量等资源。这些资源在上下文切换时是不需要修改的。
- 另外，线程也有自己的私有数据，比如栈和寄存器等，这些在上下文切换时也是需要保存的。

##### 发生线程上下文切换的场景

1. 前后两个线程属于不同进程。此时，因为资源不共享，所以切换过程就跟进程上下文切换是一样。
2. 前后两个线程属于同一个进程。此时，因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据

**中断上下文切换**

为了快速响应硬件的事件，中断处理会打断进程的正常调度和执行，转而调用中断处理程序，响应设备事件。而在打断其他进程时，就需要将进程当前的状态保存下来，这样在中断结束后，进程仍然可以从原来的状态恢复运行。

跟进程上下文不同，中断上下文切换并不涉及到进程的用户态。所以，即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等用户态资源。中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等。

对同一个 CPU 来说，中断处理比进程拥有更高的优先级，所以中断上下文切换并不会与进程上下文切换同时发生。同样道理，由于中断会打断正常进程的调度和执行，所以大部分中断处理程序都短小精悍，以便尽可能快的执行结束。

另外，跟进程上下文切换一样，中断上下文切换也需要消耗 CPU，切换次数过多也会耗费大量的 CPU，甚至严重降低系统的整体性能。所以，当你发现中断次数过多时，就需要注意去排查它是否会给你的系统带来严重的性能问题。

**使用举例**

模拟上下文切换环境

```shell
# sysbench --threads=10 --max-time=300 threads run
```

通过vmstat可以看到当前系统的上下文切换数据：

```shell
# vmstat  2 20
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 8  0      0 7430044   2084 400096    0    0     7     3   76  724  1  0 99  0  0
 7  0      0 7430044   2084 400128    0    0     0     0 2012 1990091 25 75  0  0  0
 8  0      0 7430044   2084 400128    0    0     0     0 2011 1986266 25 75  0  0  0
 8  0      0 7430044   2084 400128    0    0     0     0 2010 1989752 24 76  0  0  0
 6  0      0 7430044   2084 400128    0    0     0     0 2010 1992883 25 75  0  0  0
 8  0      0 7430044   2084 400128    0    0     0     0 2009 1977393 25 75  0  0  0
 6  0      0 7430044   2084 400128    0    0     0     0 2006 1938503 24 76  0  0  0
```

* cs（context switch）是每秒上下文切换的次数。

* in（interrupt）则是每秒中断的次数。

* r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。

* b（Blocked）则是处于不可中断睡眠状态的进程数。

如果想要查看上下文切换的详细信息需要用pidstat，由于sysbench是基于线程做的上下文切换，需要加 -t 选项：

```shell
# pidstat -wt  1
12:28:22 AM   UID      TGID       TID   cswch/s nvcswch/s  Command
...
12:28:23 AM     0     11434         -      5.00      0.00  kworker/1:1
12:28:23 AM     0         -     11434      5.00      0.00  |__kworker/1:1
12:28:23 AM     0         -     11446  41801.00 155379.00  |__sysbench
12:28:23 AM     0         -     11447  39321.00 154653.00  |__sysbench
12:28:23 AM     0         -     11448  41284.00 151069.00  |__sysbench
12:28:23 AM     0         -     11449  48000.00 136218.00  |__sysbench
12:28:23 AM     0         -     11450  29896.00 180478.00  |__sysbench
12:28:23 AM     0         -     11451  37908.00 146382.00  |__sysbench
12:28:23 AM     0         -     11452  29971.00 168312.00  |__sysbench
12:28:23 AM     0         -     11453  45331.00 142427.00  |__sysbench
12:28:23 AM     0         -     11454  26810.00 166178.00  |__sysbench
12:28:23 AM     0         -     11455  38822.00 163939.00  |__sysbench
```

* cswch ，表示每秒自愿上下文切换（voluntary context switches）的次数。

* nvcswch ，表示每秒非自愿上下文切换（non voluntary context switches）的次数。

上面两个参数的大小意味着如下的性能问题：

* 所谓自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
* 而非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

下面是如何查看中断的数据：

```shell
# cat /proc/interruptsge
...
RES:     917379     927351   Rescheduling interrupts
...
```

### CPU 使用率

**相关指标**

* user(通常缩写为us)，代表用户态CPU时间。注意，它不包括下面的nice时间，但包括了guest时间。
* nice(通常缩写为ni)，代表低优先级用户态CPU时间，也就是进程的nice值被调整为1-19之间时的CPU时间。nice可取值范围是-20到19，数值越大，优先级反而越低。
* systen(通常缩写为sys)，代表内核态CPU时间。
* idle(通常缩写为id)，代表空闲时间。注意，它不包括等待I/O的时间(iowait)。
* iowait(通常缩写为wa)，代表等待I/O的CPU时间。
* irq(通常缩写为hi)，代表处理硬中断的CPU时间。
* softirq(通常缩写为si)，代表处理软中断的CPU时间。
* steal(通常缩写为st)，代表当前系统运行在虚拟机中的时候，被其它虚拟机占用的CPU时间。
* guest(通常缩写为guest)，代表通过虚拟化运行其它操作系统的时间，也就是运行虚拟机的CPU时间。
* guest_nice(通常缩写为gnice)，代表低优先级运行虚拟机的时间。

**进程状态**

* R：Running或Runnable 的缩写，表示进程在CPU的就绪队列中，正在运行或者正在等待运行。
* D：Disk Sleep的缩写，是不可中断状态睡眠（Uninterruptiable Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其它进程或者中断打断。
* Z：Zombie的缩写，表示僵尸进程，也就是进程实际上已经结束，但是父进程还没有回收它的资源。
* S：Interruptiable Sleep缩写，是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入R状态。
* I：是Idle的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用D表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用Idle正是为了区分这种情况。要注意，D状态的进程会导致平均负载升高，I 状态的进程缺不会。
* T或t：Stopped或Traced缩写，表示进程处于暂停或者跟踪状态。
* X：就是Dead的缩写，表示进程已经消亡，不会在ps或者top命令中看到。

**案例一举例**

运行模拟故障环境

```shell
# docker run --name nginx -p 10000:80 -itd feisky/nginx
# docker run --name phpfpm -itd --network container:nginx feisky/php-fpm
```

验证环境启动是否正常

```shell
# curl  http://192.168.120.88:10000
# ab -c 10 -n 100 http://192.168.120.88:10000/  
...
Requests per second:    24.69 [#/sec] (mean) #平均请求太低
Time per request:       405.089 [ms] (mean)
...
# while true ; do ab -c 10 -n 100 http://192.168.120.88:10000/ ;done
```

查看机器负载：

```shell
%Cpu0  : 96.7 us,  0.0 sy,  0.0 ni,  3.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 97.3 us,  0.3 sy,  0.0 ni,  0.7 id,  0.0 wa,  0.0 hi,  1.7 si,  0.0 st
KiB Mem :  7990288 total,  5941568 free,   371696 used,  1677024 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7201880 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                       
 22578 bin       20   0  336684   9372   1700 R  41.2  0.1   0:13.45 php-fpm                                                       
 22580 bin       20   0  336684   9372   1700 R  40.5  0.1   0:13.45 php-fpm                                                       
 22581 bin       20   0  336684   9368   1696 R  38.5  0.1   0:13.64 php-fpm                                                       
 22582 bin       20   0  336684   9368   1696 R  38.2  0.1   0:13.20 php-fpm                                                       
 22579 bin       20   0  336684   9372   1700 R  35.5  0.1   0:13.79 php-fpm  
```

生成报告：

```shell
# perf record  -g -p 22580 
```

* perf top：类是top 能够实时显示占用CPU时钟最多的函数或者指令，可以用来查找热点函数。
* perf record：将运行命令产生的结果保存到 perf.data 中。
* perf report：读取 perf.data 数据。
* -g：开启调用关系采样，方便根据调用链来分析性能问题。

分析报告：

```shell
# docker cp perf.data phpfpm:/tmp/
# docker exec -it phpfpm /bin/bash
# apt-get update && apt-get install -y linux-perf linux-tools procps
/tmp# perf_4.9  report
...
  Children      Self  Command  Shared Object               Symbol   
-   95.11%     3.58%  php-fpm  php-fpm                     [.] execute_ex 
   - 91.53% execute_ex  
      - 18.08% 0x8c4a7c 
           5.06% sqrt  
           0.71% 0x681d21 
           0.69% 0x681d1a
      - 16.03% 0x98dea3  
         - 3.15% 0x98dd97   
              3.13% add_function  
           1.89% 0x98de23  
           0.66% add_function  
           0.54% 0x98de11 
        3.30% 0x94ede0
...
```

* Children：该符号调用的其它符号（可以理解为子函数包括直接和间接调用）占用的比例之和。
* Self：最后一列的符号（可以理解为函数）本身所占比例。
* Overhead列：该符号性能事件在所有采样中的比例，用百分比表示。
* Shard列：该函数或指令所在的动态共享对象，如内核、进程名、动态链接库名、内核模块名等。
* Object列：动态共享对象的类型。比如[.]表示用户空间的可执行程序、或者动态链接库，而[k]则表示内核空间。
* Symbol列：符号名，也就是函数名。当函数未知时，用十六进制的地址来表示。

查看容器内程序：

```shell
root@da7e0a02e36f:/tmp# grep sqrt /app/*
/app/index.php:  $x += sqrt($x);
root@da7e0a02e36f:/tmp# cat /app/index.php 
<?php
// test only.
$x = 0.0001;
for ($i = 0; $i <= 1000000; $i++) {
  $x += sqrt($x);
}

echo "It works!"
?>
```

**案例二举例**

运行案例模拟环境：

```shell
# docker run --name nginx -p 10000:80 -itd feisky/nginx:sp
# docker run --name phpfpm -itd --network container:nginx feisky/php-fpm:sp
```

验证环境：

```shell
# ab -c 100 -n 1000 http://192.168.120.88:10000/
...
Requests per second:    155.53 [#/sec] (mean)
Time per request:       642.954 [ms] (mean)
...
# ab -c 5 -t 600 http://192.168.120.88:10000/   # 设置请求时长保证Nginx端压力
```

查看系统状况：

```
%Cpu0  : 72.9 us, 21.6 sy,  0.0 ni,  5.2 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu1  : 64.2 us, 24.3 sy,  0.0 ni,  2.0 id,  0.0 wa,  0.0 hi,  9.5 si,  0.0 st

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                       
 24258 101       20   0   33092   2200    776 S   7.3  0.0   0:02.58 nginx                                                         
 30412 bin       20   0  336684   9576   1900 S   3.3  0.1   0:00.67 php-fpm                                                       
 30393 bin       20   0  336684   9580   1904 S   3.0  0.1   0:00.65 php-fpm                                                       
 30388 bin       20   0  336684   9588   1912 S   2.7  0.1   0:00.66 php-fpm                                                       
 30402 bin       20   0  336684   9576   1900 S   2.7  0.1   0:00.66 php-fpm                                                       
 30420 bin       20   0  336684   9576   1900 S   2.7  0.1   0:00.59 php-fpm                                                       
 19589 root      20   0  669588 101760  32416 S   2.0  1.3   0:49.35 dockerd                                                       
 24204 root      20   0  113364  11556   3268 S   2.0  0.1   0:00.69 containerd-shim                                               
    14 root      20   0       0      0      0 S   0.7  0.0   0:14.44 ksoftirqd/1   
```

```

05:32:23 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
05:32:24 PM     0        14    0.00    0.99    0.00    0.99     1  ksoftirqd/1
05:32:24 PM     0     19589    0.00    1.98    0.00    1.98     0  dockerd
05:32:24 PM     0     24204    0.00    1.98    0.00    1.98     1  containerd-shim
05:32:24 PM   101     24258    0.00    5.94    0.00    5.94     1  nginx
05:32:24 PM     1     30388    0.99    1.98    0.00    2.97     1  php-fpm
05:32:24 PM     1     30393    0.00    1.98    0.00    1.98     0  php-fpm
05:32:24 PM     1     30402    0.00    1.98    0.00    1.98     0  php-fpm
05:32:24 PM     1     30412    0.00    1.98    0.00    1.98     0  php-fpm
05:32:24 PM     1     30420    0.99    2.97    0.00    3.96     1  php-fpm
05:32:24 PM     0     62993    0.00    0.99    0.00    0.99     1  pidstat
```

通过 top 和pidstat 命令可以看到系统中运行的程序负载都正常，但是CPU使用率都在60%+，需要找到占用资源的程序。

这边通过 ps 命令看到系统很多stress程序

```shell
bin      121057  0.0  0.0   4280   628 pts/0    S+   17:55   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
bin      121058  0.0  0.0   4280   632 pts/0    S+   17:55   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
bin      121059  0.0  0.0   7272   424 pts/0    R+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121060  0.0  0.0   4280   628 pts/0    S+   17:55   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
bin      121061  0.0  0.0   7272   428 pts/0    S+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121062  0.0  0.0   4280   632 pts/0    S+   17:55   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
bin      121063  0.0  0.0   7272   428 pts/0    S+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121064  0.0  0.0   4280   628 pts/0    S+   17:55   0:00 sh -c /usr/local/bin/stress -t 1 -d 1 2>&1
bin      121065  0.0  0.0      0     0 pts/0    Z+   17:55   0:00 [stress] <defunct>
bin      121066  0.0  0.0   7272   424 pts/0    R+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121067  0.0  0.0      0     0 pts/0    Z+   17:55   0:00 [stress] <defunct>
bin      121068  0.0  0.0   8176   332 pts/0    R+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121069  0.0  0.0   7272   100 pts/0    R+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121070  0.0  0.0   7272   428 pts/0    S+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
bin      121071  0.0  0.0   8176   596 pts/0    R+   17:55   0:00 /usr/local/bin/stress -t 1 -d 1
```

通过查找stress进程号查看应用详情还是找不到该进程信息：

```
# pidstat -p 121070 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/05/2021      _x86_64_        (2 CPU)

05:56:21 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
```

目前情况是：系统一直运行stress程序，通过pidstat无法查找到程序的详细信息。说明程序在不停的重启，造成这种情况有如下俩个原因：

* 进程在不停地崩溃重启，比如因为段错误、配置错误等等。这时进程在退出后可能又被监控系统自动重启了。
* 这些进程都是短时进程，也就是在其他应用内部通过exec调用的外面命令。这些命令一般都只运行很短的时间就会结束，你很难用top这种间隔时间比较长的工具发现。

子进程和父进程的关系可以通过pstress查看：

```shell
# yum install -y psmisc
# pstree  |grep stress
        |-containerd-shim-+-php-fpm-+-2*[php-fpm---sh---stress---stress]
        |                 |         |-2*[php-fpm---sh---stress]
```

查看应用中stress信息：

```
# docker exec -it phpfpm /bin/bash
root@829734352b90:/app# grep stress * 
index.php:// fake I/O with stress (via write()/unlink()).
index.php:$result = exec("/usr/local/bin/stress -t 1 -d 1 2>&1", $output, $status);
root@829734352b90:/app# cat index.php 
<?php
// fake I/O with stress (via write()/unlink()).
$result = exec("/usr/local/bin/stress -t 1 -d 1 2>&1", $output, $status);
if (isset($_GET["verbose"]) && $_GET["verbose"]==1 && $status != 0) {
  echo "Server internal error: ";
  print_r($output);
} else {
  echo "It works!";
}
```

通过上面的信息可以使用如下请求：

```
# curl http://192.168.120.88:10000?verbose=1
Server internal error: Array
(
    [0] => stress: info: [25103] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd
    [1] => stress: FAIL: [25104] (563) mkstemp failed: Permission denied
    [2] => stress: FAIL: [25103] (394) <-- worker 25104 returned error 1
    [3] => stress: WARN: [25103] (396) now reaping child worker processes
    [4] => stress: FAIL: [25103] (400) kill error: No such process
    [5] => stress: FAIL: [25103] (451) failed run completed in 0s
)
```

通过 perf 命令查看系统运行状态：

```
# perf record -ag -- sleep 2 
# perf  report
  Children      Self  Command          Shared Object            Symbol   
+    8.25%     0.00%  stress           [kernel.kallsyms]        [k] irq_exit  
+    8.25%     0.00%  stress           [kernel.kallsyms]        [k] do_softirq
+    8.25%     0.00%  stress           [kernel.kallsyms]        [k] call_softirq
+    8.25%     6.42%  stress           [kernel.kallsyms]        [k] __do_softirq   
+    7.52%     0.01%  stress           [kernel.kallsyms]        [k] page_fault 
+    7.51%     0.00%  stress           [kernel.kallsyms]        [k] do_page_fault
+    7.04%     2.02%  stress           [kernel.kallsyms]        [k] __do_page_fault 
+    6.71%     0.00%  stress           [kernel.kallsyms]        [k] apic_timer_interrupt
+    6.71%     0.00%  stress           [kernel.kallsyms]        [k] smp_apic_timer_interrupt 
+    4.72%     0.32%  stress           [kernel.kallsyms]        [k] handle_mm_fault  
+    4.03%     0.00%  php-fpm          [unknown]                [.] 0x6cb6258d4c544155
```

也可以使用execsnoop工具查看：

```
# git clone --depth 1 https://github.com/brendangregg/perf-tools
perf-tools]# ./execsnoop 
Tracing exec()s. Ctrl-C to end.
Instrumenting sys_execve
   PID   PPID ARGS
 68597  68593 gawk -v o=1 -v opt_name=0 -v name= -v opt_duration=0 [...]
 68598  68596 cat -v trace_pipe
 ...
 68936  68934 /usr/local/bin/stress -t 1 -d 1
 68939  68938 /usr/local/bin/stress -t 1 -d 1
 68937  68935 /usr/local/bin/stress -t 1 -d 1
 ...
```

execsnoop 就是一个专为短时进程设计的工具。它通过ftrace实时监控进程的exec()行为，并输出短时进程基本信息，包括进程pid、父进程PID、命令行参数以及执行的结果。

遇到常规问题无法解释的CPU使用率情况时，首先要想到有可能是短时应用导致的问题，比如有下面两种情况：

* 应用里直接调用了其它二进制程序，这些程序通常运行时间比较短，通过top等工具也不容易发现。
* 应用本身在不停地崩溃重启，而启动过程地资源初始化，很可能会占用相当多地CPU。

**案例三举例**

```
# docker run --privileged --name=app -itd feisky/app:iowait
```

运行一段时间使用top查看系统状态：

```
top - 16:49:12 up  1:09,  2 users,  load average: 0.01, 0.02, 0.05
Tasks: 182 total,   2 running, 114 sleeping,   0 stopped,  66 zombie
%Cpu0  :  0.4 us,  3.2 sy,  0.0 ni, 67.9 id,  0.0 wa,  0.0 hi, 28.6 si,  0.0 st
%Cpu1  :  0.0 us, 10.8 sy,  0.0 ni, 70.3 id, 18.9 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  7990288 total,  7011948 free,   279892 used,   698448 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7384232 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                       
     3 root      20   0       0      0      0 S  21.9  0.0   0:09.06 ksoftirqd/0                                                   
 20204 root      20   0       0      0      0 Z  16.3  0.0   0:00.49 app                                                           
 20205 root      20   0       0      0      0 Z  11.6  0.0   0:00.35 app                                                           
  8878 root      20   0   90392   3212   2348 S   0.7  0.0   0:02.68 rngd    
```

通过观察有两个重要的异常：

* 僵尸进程一直在增加，这种一般是应用程序没有正确的清理子进程。
* 即使是ssd磁盘也会出现iowait的情况。

可以通过dstat命令查看当前系统的磁盘io：

```
# dstat 
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
  0   1  99   0   0   0|  23M   20k|   0     0 |   0     0 | 110   133 
  0   1 100   0   0   0|   0     0 | 180B 6296B|   0     0 |  97   141 
  0  15  74   0   0  11| 623M    0 |  60B  390B|   0     0 |1012   256 
  0  22  52   0   0  25|1937M    0 |  60B  390B|   0     0 |3404   402 
  1   0  99   0   0   0|   0     0 | 180B 6446B|   0     0 | 105   159 
```

在发现磁盘读很高，在查询通过top中命令发现的异常进程ID状态：

```
# pidstat -d -p 20204 1 3
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/09/2021      _x86_64_        (2 CPU)

04:50:29 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
04:50:30 PM     0     20204      0.00      0.00      0.00  app
04:50:31 PM     0     20204      0.00      0.00      0.00  app
04:50:32 PM     0     20204      0.00      0.00      0.00  app
Average:        0     20204      0.00      0.00      0.00  app
# pidstat -d  1 10
04:51:29 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

04:51:30 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
04:51:31 PM     0     20271 786432.00      0.00      0.00  app
04:51:31 PM     0     20272 1230335.50      0.00      0.00  app

04:51:31 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
04:51:32 PM     0     20271 524288.00      0.00      0.00  app
04:51:32 PM     0     20272  80384.50      0.00      0.00  app

```

```
# strace -p 20272 
strace: attach: ptrace(PTRACE_ATTACH, ...): Operation not permitted
```

```
# ps aux |grep 20272
root      20272  0.4  0.0      0     0 pts/0    Z+   16:51   0:00 [app] <defunct>
root      20325  0.0  0.0 112712   976 pts/1    S+   16:53   0:00 grep --color=auto 20272
```

在上面使用strace查看进程运行的详细状态失败后，可以使用perf观察当前系统的进程运行信息：

```
# perf record -g
# perf report 
...
-   15.86%     0.00%  app             [kernel.kallsyms]    [k] sys_read 
     sys_read 
     vfs_read  
     do_sync_read 
     blkdev_aio_read   
   - generic_file_aio_read   
      - 15.85% blkdev_direct_IO
         - __blockdev_direct_IO   
            - 15.74% do_blockdev_direct_IO 
               + 6.36% submit_bio    
               + 3.37% put_page  
                 2.03% __get_page_tail  
               + 1.29% dio_bio_complete   
               + 1.03% get_user_pages_fast 
+   15.86%     0.00%  app             [kernel.kallsyms]    [k] vfs_read  
+   15.86%     0.00%  app             [kernel.kallsyms]    [k] do_sync_read 
+   15.86%     0.00%  app             [kernel.kallsyms]    [k] blkdev_aio_read
```

通perf采集数据可以观察到进程是正在对磁盘进行直接读，这种方式是绕过了系统缓存，每个读请求都会从磁盘直接读。

下面先修复读数据的参数在运行程序进行观察：

```
# docker rm -f app
# docker run --privileged --name=app -itd feisky/app:iowait-fix1
```

运行top 发现系统还是在生成僵尸进程：

```
top - 17:51:06 up  2:11,  2 users,  load average: 0.02, 0.05, 0.05
Tasks: 144 total,   1 running, 115 sleeping,   0 stopped,  28 zombie
%Cpu0  :  0.0 us,  2.9 sy,  0.0 ni, 97.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  2.5 sy,  0.0 ni, 97.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  7990288 total,  5565036 free,   277372 used,  2147880 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7382368 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                       
 21646 root      20   0       0      0      0 Z   9.0  0.0   0:00.27 app                                                           
 21647 root      20   0       0      0      0 Z   9.0  0.0   0:00.27 app  
```

在通过pstree定位僵尸进程的父进程还是app，进入容器查询资源释放段代码进行修改。

```
# pstree  -aps 21646  # -a 输出命令行选项 -s 指定进程的父进程
systemd,1 --switched-root --system --deserialize 22
  └─containerd-shim,21558 -namespace moby -id bae00a1c8590e3fcd803dc503bf5e6a0bf0ed2b0daec25caa45d9615dee6d883 -address/run/
      └─app,21578
          └─(app,21646)
# ps aux |grep 21578
root      21578  0.0  0.0   4500   588 pts/0    Ss+  17:49   0:00 /app
root      21675  0.0  0.0 112712   976 pts/1    S+   17:52   0:00 grep --color=auto 21578
```

### Linux 中断

Linux 将中断处理过程分成了两个阶段，上半部和下半部：

* 上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行。
* 下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。在系统中中名字为"ksoftirqd/CPU编号"。网络的收发、定时、内核中的调度和RCU锁等都属于软中断。

**查看软中断和内核线程**

* /proc/softirqs 提供了软中断运行情况
* /proc/interrupts 提供了硬中断运行情况

**案例一分析**

运行一个nginx容器并测试80端口：

```
# docker run -itd --name=nginx -p 80:80 nginx
# curl  -I http://192.168.120.88
HTTP/1.1 200 OK
...
```

在另外一台机器执行hping3命令，模拟nginx的客户端请求：

```
# -S 表示设置TCP协议的SYN； -p 表示目的端口； -i u100 表示每隔100微秒发送一个网络帧。
# hping3 -S -p 80 -i u100 192.168.120.88 &>/dev/null 
```

在运行nginx主机执行top查看系统状态：

```
top - 22:15:49 up  6:35,  2 users,  load average: 0.05, 0.04, 0.05
Tasks: 118 total,   2 running, 116 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni, 70.1 id,  0.0 wa,  0.0 hi, 29.9 si,  0.0 st
KiB Mem :  7990288 total,  5344052 free,   304708 used,  2341528 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7313200 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                       
    14 root      20   0       0      0      0 R   2.3  0.0   0:02.83 ksoftirqd/1                                                   
     9 root      20   0       0      0      0 S   1.0  0.0   0:05.64 rcu_sched                                                     
  8878 root      20   0   90392   3212   2348 S   0.7  0.0   0:06.97 rngd                                                          
 19743 root      20   0  446172  46416  16436 S   0.3  0.6   2:16.14 containerd                                                    
 22282 root      20   0  164072   2260   1600 R   0.3  0.0   0:00.12 top                                                           
     1 root      20   0   43512   3816   2552 S   0.0  0.0   0:02.27 systemd                                                       
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.06 kthreadd                                                      
     3 root      20   0       0      0      0 S   0.0  0.0   1:17.31 ksoftirqd/0                                                   
     5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H                                                  
     7 root      rt   0       0      0      0 S   0.0  0.0   0:01.16 migration/0                                                   
     8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh         
```

通过观察最主要的问题就是系统的软中断占2.3%的CPU，如果想查看系统的中断状况通过如下命令：

```
# 一核CPU
# watch -d "/bin/cat /proc/softirqs | /usr/bin/awk 'NR == 1{printf \"%13s %s\n\",\" \",\$1}; NR > 1{printf \"%13s %s\n\",\$1,\$2}'" 
# 二核CPU
# watch -d "/bin/cat /proc/softirqs | /usr/bin/awk 'NR == 1{printf \"%13s %10s %10s\n\",\" \",\$1,\$2}; NR > 1{printf \"%13s %10s %10s\n\",\$1,\$2,\$3}'"

                    CPU0       CPU1
          HI:          0          1
       TIMER:     407741     722416   # 定时中断
      NET_TX:       5204         18   # 网络发送
      NET_RX:         57    2462141   # 网络接收
       BLOCK:     283991          0
BLOCK_IOPOLL:          0          0
     TASKLET:       3161        973
       SCHED:     210527     333100   # 内核调度
     HRTIMER:          0          0
         RCU:     200242     321339   # RCU锁
```

通过上面的数据可以明显的看到网络接收数据包的软中断增长很高，通过mpstat 和sar 都能查看网络状态：

```
# mpstat -I SUM -P ALL 5 
10:37:52 PM  CPU    intr/s
10:37:57 PM  all   5154.40
10:37:57 PM    0    154.60
10:37:57 PM    1   4999.00

10:37:57 PM  CPU    intr/s
10:38:02 PM  all   5487.80
10:38:02 PM    0    149.60
10:38:02 PM    1   5338.20
```

下面主要看下sar的输出数据：

```
# sar -n DEV 1
10:16:22 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
10:16:23 PM veth664b507   2731.00   5464.00    154.69    288.14      0.00      0.00      0.00
10:16:23 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:16:23 PM     ens33   5464.00   2732.00    320.16    160.62      0.00      0.00      0.00
10:16:23 PM   docker0   2731.00   5465.00    117.35    288.19      0.00      0.00      0.00

10:16:23 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
10:16:24 PM veth664b507   3149.00   6296.00    178.36    332.02      0.00      0.00      0.00
10:16:24 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:16:24 PM     ens33   6298.00   3150.00    369.02    185.11      0.00      0.00      0.00
10:16:24 PM   docker0   3149.00   6295.00    135.31    331.96      0.00      0.00      0.00

Average:        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
Average:    veth664b507   2950.00   5900.09    167.09    311.14      0.00      0.00      0.00
Average:           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:        ens33   5901.58   2951.93    345.80    174.66      0.00      0.00      0.00
Average:      docker0   2950.09   5900.09    126.76    311.14      0.00      0.00      0.00
```

* rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是PPS。
* rxkB/s 和 txkB/s 分别表示每秒接受、发送的千字节数，也就是BPS。

数据分析网卡信息：接收的平均PPS为5900，而接收的BPS为311KB。实际接收网络帧为：311*1024/5900约等于54字节。说明接收的都是小包。在运行tcpdump进行抓包分析：

```
# tcpdump -i ens33 -n tcp port 80
...
22:17:53.602019 IP 192.168.120.74.64695 > 192.168.120.88.http: Flags [S], seq 1218751833, win 512, length 0
22:17:53.602070 IP 192.168.120.74.64694 > 192.168.120.88.http: Flags [R], seq 329855422, win 0, length 0
22:17:53.602223 IP 192.168.120.88.http > 192.168.120.74.64695: Flags [S.], seq 443833703, ack 1218751834, win 29200, options [mss 1460], length 0
```

## 性能优化思路

### 性能优化方法论

优化前需要思考下面三个问题，如果有了答案在开始优化。

* 首先，既然要做性能优化，那要怎么判断它是不是有效呢？特别是优化后，到底能提升多少性能呢？
* 第二，性能问题通常不是独立的，如果有多个性能问题同时发生，你应该先优化哪一个呢？
* 第三，提升性能的方法并不是唯一的，当有多种方法可以选择时，你会选用哪一种呢？是不是总选那个最大程度提升性能的方法就行了呢？ 

### 评估性能优化的效果

* 确定性能的量化指标。
* 测试优化前的性能指标。
* 测试优化后的性能指标。 

### 多个性能问题同时存在的优化思路

动手优化之前先动脑，先把所有这些性能问题给分析一遍，找出最重要的、可以最大程度提升性能的问题，从它开始优化。这样的好处是，不仅性能提升的收益最大，而且很可能其他问题都不用优化，就已经满足了性能要求。 

那关键就在于，怎么判断出哪个性能问题最重要。这其实还是我们性能分析要解决的核心问题，只不过这里要分析的对象，从原来的一个问题，变成了多个问题，思路其实还是一样的。

所以，你依然可以用前面讲过的方法挨个分析，分别找出它们的瓶颈。分析完所有问题后，再按照因果等关系，排除掉有因果关联的性能问题。最后，再对剩下的性能问题进行优化。 

### 应用程序优化

* 编译器优化：很多编译器都会提供优化选项，适当开启它们，在编译阶段你就可以获得编译器的帮助，来提升性能。比如， gcc 就提供了优化选项 -O2，开启后会自动对应用程序的代码进行优化。
* 算法优化：使用复杂度更低的算法，可以显著加快处理速度。比如，在数据比较大的情况下，可以用 O(nlogn) 的排序算法（如快排、归并排序等），代替 O(n^2) 的排序算法（如冒泡、插入排序等）。
* 异步处理：使用异步处理，可以避免程序因为等待某个资源而一直阻塞，从而提升程序的并发处理能力。比如，把轮询替换为事件通知，就可以避免轮询耗费 CPU 的问题。
* 多线程代替多进程：前面讲过，相对于进程的上下文切换，线程的上下文切换并不切换进程地址空间，因此可以降低上下文切换的成本。
* 善用缓存：经常访问的数据或者计算过程中的步骤，可以放到内存中缓存起来，这样在下次用时就能直接从内存中获取，加快程序的处理速度。 

###  系统优化 

* CPU 绑定：把进程绑定到一个或者多个 CPU 上，可以提高 CPU 缓存的命中率，减少跨 CPU 调度带来的上下文切换问题。
* CPU 独占：跟 CPU 绑定类似，进一步将 CPU 分组，并通过 CPU 亲和性机制为其分配进程。这样，这些 CPU 就由指定的进程独占，换句话说，不允许其他进程再来使用这些 CPU。
* 优先级调整：使用 nice 调整进程的优先级，正值调低优先级，负值调高优先级。优先级的数值含义前面我们提到过，忘了的话及时复习一下。在这里，适当降低非核心应用的优先级，增高核心应用的优先级，可以确保核心应用得到优先处理。
* 为进程设置资源限制：使用 Linux cgroups 来设置进程的 CPU 使用上限，可以防止由于某个应用自身的问题，而耗尽系统资源。
* NUMA（Non-Uniform Memory Access）优化：支持 NUMA 的处理器会被划分为多个 node，每个 node 都有自己的本地内存空间。NUMA 优化，其实就是让 CPU 尽可能只访问本地内存。
* 中断负载均衡：无论是软中断还是硬中断，它们的中断处理程序都可能会耗费大量的 CPU。开启 irqbalance 服务或者配置 smp_affinity，就可以把中断处理过程自动负载均衡到多个 CPU 上。

###  千万避免过早优化 

 “过早优化是万恶之源” ， 一方面，优化会带来复杂性的提升，降低可维护性；另一方面，需求不是一成不变的。针对当前情况进行的优化，很可能并不适应快速变化的新需求。这样，在新需求出现时，这些复杂的优化，反而可能阻碍新功能的开发。 性能优化最好是逐步完善，动态进行，不追求一步到位，而要首先保证能满足当前的性能要求。当发现性能不满足要求或者出现性能瓶颈时，再根据性能评估的结果，选择最重要的性能问题进行优化。 





