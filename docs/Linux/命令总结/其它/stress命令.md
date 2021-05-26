## stress

stress 是一款施加负荷和压力测试系统的工具。

### 选项

* --version：显示版本号
* -v, --verbose：显示详细的信息
* -q, --quiet：程序在运行的过程中不输出信息
* -n, --dry-run：输出程序会做什么而并不实际执行相关的操作
* -t, --timeout N ：在 N 秒后结束程序  
* --backoff N：等待N微妙后开始运行
* -c, --cpu N： 产生 N 个进程，每个进程都反复不停的计算随机数的平方根
* -i, --io N：产生 N 个进程，每个进程反复调用 sync() 将内存上的内容写到硬盘上
* -m, --vm N：产生 N 个进程，每个进程不断分配和释放内存
* --vm-bytes B：指定分配内存的大小
* --vm-stride B：不断的给部分内存赋值，让 COW(Copy On Write)发生
* --vm-hang N： 指示每个消耗内存的进程在分配到内存后转入睡眠状态 N 秒，然后释放内存，一直重复执行这个过程
* --vm-keep：一直占用内存，区别于不断的释放和重新分配(默认是不断释放并重新分配内存)
* -d, --hdd N： 产生 N 个不断执行 write 和 unlink 函数的进程(创建文件，写入内容，删除文件)
* --hdd-bytes B：指定文件大小

### 安装

```shell
# cat /etc/redhat-release 
CentOS Linux release 7.4.1708 (Core)
# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
# yum install -y stress
```

### 使用举例

#### 消耗CPU资源

```shell
# stress -c 4
stress: info: [12244] dispatching hogs: 4 cpu, 0 io, 0 vm, 0 hdd
# top
...
%Cpu0  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12412 root      20   0    7308    100      0 R  50.3  0.0   0:17.46 stress                                  
12411 root      20   0    7308    100      0 R  50.0  0.0   0:17.35 stress                                  
12413 root      20   0    7308    100      0 R  49.7  0.0   0:17.49 stress                                  
12414 root      20   0    7308    100      0 R  49.7  0.0   0:17.45 stress 
...
```

#### 消耗内存资源

--vm-keep 和 --vm-hang 都可以用来模拟只有少量内存的机器，但是指定它们时 CPU 的使用情况是不一样的。

```shell
# stress --vm 2 --vm-bytes 500M --vm-keep
stress: info: [12389] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
# top
...
%Cpu0  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 99.7 us,  0.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12424 root      20   0  519312 512196    128 R  99.7  6.4   0:15.42 stress                                  
12425 root      20   0  519312 512196    128 R  99.3  6.4   0:15.32 stress  
```

```shell
# stress --vm 2 --vm-bytes 500M --vm-hang 5
stress: info: [12368] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
# top
...
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.3 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12416 root      20   0  519312 437452    132 R   5.3  5.5   0:00.95 stress                                  
12417 root      20   0  519312 390348    132 R   4.7  4.9   0:00.95 stress   
...
```

--vm-stride B：不断的给部分内存赋值，让 COW(Copy On Write)发生。只要指定了内存相关的选项，这个操作就会执行，只是大小为默认的 4096。其中bytes 为消耗的总内存大小，stride 为间隔。

该参数会影响 CPU 状态 us 和 sy：

```shell
# stress --vm 2 --vm-bytes 500M --vm-stride 64
stress: info: [12398] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
# top
...
%Cpu0  : 42.3 us, 57.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 41.7 us, 58.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12427 root      20   0  519312 409884    128 R 100.0  5.1   0:39.65 stress                                  
12428 root      20   0  519312 512080    128 R  99.3  6.4   0:39.38 stress  
```

```shell
# stress --vm 2 --vm-bytes 500M --vm-stride 1M
stress: info: [12429] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
# top
...
%Cpu0  :  0.0 us,100.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.7 us, 99.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12430 root      20   0  519312  47328    128 R  99.7  0.6   0:31.18 stress                                  
12431 root      20   0  519312  26848    128 R  99.0  0.3   0:31.01 stress    
```

原因是单独的赋值和对比操作可以让 CPU 在用户态的负载占到 99% 以上。--vm-stride 值增大就意味着减少赋值和对比操作，这样就增加了内存的释放和分配次数(cpu在内核空间的负载)。

不指定 --vm-stride 选项就使用默认值是 4096，CPU 负载情况居于前两者之间：

```shell
# stress --vm 2 --vm-bytes 500M
stress: info: [12438] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
# top
...
%Cpu0  :  5.3 us, 94.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  4.7 us, 95.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12439 root      20   0  519312 162644    128 R 100.0  2.0   0:54.63 stress                                  
12440 root      20   0  519312 189268    128 R  99.7  2.4   0:54.27 stress  
```

### 消耗IO资源

```shell
# stress -i 4
stress: info: [12447] dispatching hogs: 0 cpu, 4 io, 0 vm, 0 hdd
# top
...
%Cpu0  :  1.0 us, 99.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  1.4 us, 94.6 sy,  0.0 ni,  4.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12450 root      20   0    7308    100      0 R  65.1  0.0   2:04.91 stress                                  
12448 root      20   0    7308    100      0 R  36.5  0.0   2:04.22 stress                                  
12449 root      20   0    7308    100      0 R  36.5  0.0   2:02.09 stress                                  
12451 root      20   0    7308    100      0 R  35.9  0.0   2:07.00 stress 
```

### 压测磁盘和IO

```shell
# stress -d 1 --hdd-bytes 10M
stress: info: [12470] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd
# top
...
%Cpu0  :  0.0 us, 56.9 sy,  0.0 ni, 42.8 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu1  :  0.0 us, 43.5 sy,  0.0 ni, 56.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                 
12471 root      20   0    8212   1128     36 R 100.0  0.0   1:09.79 stress  
```

### 组合使用

```shell
stress --cpu 8 --io 4 --vm 2 --vm-bytes 128M --timeout 10s
```



<https://www.cnblogs.com/sparkdev/p/10354947.html>