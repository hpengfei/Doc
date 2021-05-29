## 基础概念

### Linux文件系统的组成

* 索引解点：简称 inode，用来记录文件的元数据，比如 inode 编号、文件大小、访问权限、修改日期、数据的位置等。索引节点和文件一一对应，它跟文件内容一样，都会被持久化存储到磁盘中。所以索引节点同样占用磁盘空间。
* 目录项：简称为 dentry，用来记录文件的名字、索引节点指针以及与其它目录项的关联关系。多个关联的目录项就构成了文件系统的目录结构。不过不同于索引节点，目录项是由内核维护的一个内存数据结构，所以通常被叫做目录项缓存。
* VFS：定义了一组所有文件系统都支持的数据结构和标准接口。用户进程和内核中的其它子系统，只需要跟VFS提供的统一接口进行交互就可以了，而不需要再关心底层各种文件系统的实现细节。
* 逻辑块：是由连续磁盘扇区构成的最小读写单元，用来存储文件数据。

* 超级块：用来记录文件系统整体的状态，如索引节点和逻辑块的使用情况等。

### I/O 调度

* NONE：这种不能算调度算法。因为它完全不使用任何I/O调度器，对文件系统和应用程序的I/O其实不做任何处理，常用在虚拟机中（此时磁盘I/O调度完全由物理机负责）。
* NOOP：最简单的一种I/O调度算法，它实际上是一个先入先出的队列，只做一些最基本的请求合并，常用于SSD磁盘。
* CFQ（Completely Fair Scheduler）也被称为完全公平调度器，是现在很多发行版的默认I/O调度器，它为每个进程维护了一个I/O调度队列，并按照时间片来均匀分布每个进程的I/O请求。适用于桌面环境，多媒体应用等。
* DeadLine：分别为读、写请求创建了不同的I/O队列，可以提高机械磁盘的吞吐量，并保证达到最终期限的请求被优先处理。DeadLine调度算法多用在I/O压力较重的场景，比如数据库等。

### 性能指标

* 使用率：是指磁盘处理 I/O 的时间百分比。过高的使用率（比如超过80%），通常意味着磁盘 I/O 存在性能瓶颈。
* 饱和度：是指磁盘处理I/O的繁忙程度，过高的饱和度意味着磁盘存在严重的性能瓶颈。当饱和度为100%时，磁盘无法接受新的I/O请求数。
* IOPS（Input/Output per Second）是指每秒的I/O请求数。
* 吞吐量：是指每秒的I/O请求大小。
* 响应时间：是指I/O请求从发出到收到响应的间隔时间。

一般来说我们在为应用程序的服务器选项时，要先对磁盘I/O做基准测试，以方便可以准确评估磁盘性能是否可以满足应用程序的需求。这里建议使用基准性能测试攻击fio来测试磁盘的IOPS、吞吐量以及响应时间等核心指标。

## 实际使用

### 磁盘I/0观测

```
# iostat -d -x 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/16/2021      _x86_64_        (2 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.01    0.43    0.15    17.92     1.46    66.80     0.00    0.94    0.80    1.34   0.44   0.03
```

| 性能指标 |                                             |                                                          |
| -------- | ------------------------------------------- | -------------------------------------------------------- |
| rrqm/s   | 每秒合并的读请求数                          | %rrqm表示合并读请求的百分比                              |
| wrqm/s   | 每秒合并的写请求数                          | %wrqm表示合并写请求的百分比                              |
| r/s      | 每秒发送给磁盘的读请求数                    | 合并后的请求数                                           |
| w/s      | 每秒发送给磁盘的写请求数                    | 合并后的请求数                                           |
| rkB/s    | 每秒从磁盘读取的数据量                      | 单位为kB                                                 |
| wkB/s    | 每秒向磁盘写入的数据量                      | 单位为kB                                                 |
| avgrq-sz | 平均请求队列长度                            | 旧版中为avgqu-sz                                         |
| rareq-sz | 平均读请求大小                              | 单位为kB                                                 |
| wareq-sz | 平均写请求大小                              | 单位为kB                                                 |
| await    |                                             |                                                          |
| r_await  | 读请求处理完成等待时间                      | 包括队列中的等待时间和设备实际处理的时间，单位为毫秒     |
| w_await  | 写请求处理完成等待时间                      | 包括队列中的等待时间和设备实际处理的时间，单位为毫秒     |
| svctm    | 处理I/O请求所需的平均时间（不包括等待时间） | 单位为毫秒。这是推断数据不保证完全准确。                 |
| %util    | 磁盘处理I/O的时间百分比                     | 使用率，由于可能存在并行I/O，100%并不一定表明磁盘I/O饱和 |

* %util：磁盘I/O使用率
* r/s +w/s：就是IOPS
* rkB/s +wkB/s：就是吞吐量
* r_await +w_await：就是响应时间

### 进程I/O观测

```
# pidstat -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/16/2021      _x86_64_        (2 CPU)

08:50:04 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
```

* kB_rd/s：每秒读取的数据大小，单位KB
* kB_wr/s：每秒发出写请求数据大小，单位KB
* kB_ccwr/s：每秒取消的写请求数据大小，单位KB

```
# iotop
Total DISK READ :       0.00 B/s | Total DISK WRITE :       0.00 B/s
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s
   TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
     1 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % systemd --switched-root --system --deserialize 22
     2 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [kthreadd]
```

### 案例分析一

```
# docker run -v /tmp/:/tmp  --name=app -itd feisky/logapp
```

运行完示例后先用top看下系统的情况：

```
top - 22:12:26 up  1:08,  2 users,  load average: 0.23, 0.06, 0.06
Tasks: 122 total,   2 running, 120 sleeping,   0 stopped,   0 zombie
%Cpu(s): 14.8 us, 44.1 sy,  0.0 ni, 31.3 id,  0.0 wa,  0.0 hi,  9.7 si,  0.0 st
KiB Mem :  7990288 total,  5766108 free,   581252 used,  1642928 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7075204 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 10354 root      20   0  346944 315988   3260 R  78.0  4.0   0:14.25 python
     6 root      20   0       0      0      0 S  52.0  0.0   0:20.82 kworker/u256:0
 10235 root      20   0       0      0      0 S   4.0  0.0   0:00.72 kworker/1:0
```

通过上面的数据可以看到一个 python 进程占用了不少CPU，也都在能接受的范围。下面就是通过dstat看下系统状态：

```
# dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
  1   2  97   0   0   0|  90k   25M|   0     0 |   0     0 | 169   171
 14  37  40   0   0   9|   0   428M| 120B  134B|   0     0 |1984   344
 16  37  32   0   0  16|   0   577M| 330B  760B|   0     0 |2113   307
```

通过dstat命令能很明显的发现当前系统的写入很大，既然磁盘IO大，那么久用iostat看下磁盘的详细信息：

```
# iostat -x -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/23/2021      _x86_64_        (2 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.77    1.43   20.67    92.41 10104.84   922.60     0.14    6.22    0.64    6.61   0.37   0.81

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    1.00 1601.00     8.00 819204.00  1022.74     1.27    0.79    1.00    0.79   0.41  64.90

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00  882.00     0.00 450568.50  1021.70     0.56    0.63    0.00    0.63   0.42  37.30

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00 1366.00     0.00 699392.00  1024.00     1.14    0.84    0.00    0.84   0.35  48.20
```

通过上面的dstat看到的数据，以及磁盘sda的使用率在50%+，并且请求队列的长度也在1000+。在ssd磁盘中还有这么高的写入说明当前系统有进程在做大量的写入操作。同理也可以使用pidstat查看当前系统的磁盘使用信息：

```
# pidstat -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/23/2021      _x86_64_        (2 CPU)

10:13:00 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:13:01 PM     0     10354      0.00 608316.83      0.00  python

10:13:01 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:13:02 PM     0     10354      0.00 608320.79  60835.64  python

10:13:02 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:13:03 PM     0     10354      0.00 614404.00      0.00  python
```

找到了大量写入程序的PID，然后通过strace和lsof 分析进程的信息：

```
# strace -p 10354
...
stat("/tmp/logtest.txt.1", {st_mode=S_IFREG|0644, st_size=943718535, ...}) = 0
unlink("/tmp/logtest.txt.1")            = 0
stat("/tmp/logtest.txt", {st_mode=S_IFREG|0644, st_size=943718535, ...}) = 0
rename("/tmp/logtest.txt", "/tmp/logtest.txt.1") = 0
open("/tmp/logtest.txt", O_WRONLY|O_CREAT|O_APPEND|O_CLOEXEC, 0666) = 3
fcntl(3, F_SETFD, FD_CLOEXEC)           = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
lseek(3, 0, SEEK_END)                   = 0
ioctl(3, TIOCGWINSZ, 0x7ffc53f8c940)    = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
ioctl(3, TIOCGWINSZ, 0x7ffc53f8c860)    = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fd4a53000
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fc1e52000
write(3, "2021-05-23 14:13:17,947 - __main"..., 314572844) = 314572844
munmap(0x7f5fc1e52000, 314576896)       = 0
write(3, "\n", 1)   
```

```
# lsof -p 10354
COMMAND   PID USER   FD   TYPE DEVICE  SIZE/OFF     NODE NAME
...
python  10354 root    3w   REG    8,3 629145690 18660068 /tmp/logtest.txt
```

通过上面的信息可以看到，一个进程号为 10354 的程序正在往文件/tmp/logtest.txt 中大量的写入数据。找到这个基本上就可以让开发查下代码，是不是哪个地方的写入有问题。

### 案例分析二

运行案例：

```
# docker run --name=app -p 10000:80 -itd feisky/word-pop
```

可以先测试案例是否启动，然后在另外一台机器模拟请求。发现响应时间很慢：

```
# while true; do time curl http://192.168.120.88:10000/popularity/word;sleep 1; done
{
  "popularity": 0.0,
  "word": "word"
}

real    0m12.091s
user    0m0.001s
sys     0m0.005s
```

遇到问题先看top，通过输出信息可以看到系统除了负载高，没有太多异常。

```
top - 22:14:22 up 3 min,  1 user,  load average: 0.79, 0.41, 0.18
Tasks: 124 total,   1 running, 123 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.3 us,  6.5 sy,  0.0 ni, 92.1 id,  0.0 wa,  0.0 hi,  1.0 si,  0.0 st
%Cpu1  : 34.3 us, 29.0 sy,  0.0 ni, 36.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  7990288 total,  7229756 free,   281236 used,   479296 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7397840 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 10071 root      20   0   97036  20384   4028 S  69.1  0.3   0:27.81 python
    47 root      20   0       0      0      0 S   2.3  0.0   0:02.26 kworker/u256:1
     9 root      20   0       0      0      0 S   1.0  0.0   0:01.20 rcu_sched
```

通过dstat可以明显的发现系统的磁盘写入很大，下一步要查磁盘IO的详细信息：

```
# dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
  9  10  80   0   0   1|1812k   30M|   0     0 |   0     0 | 763  1238
 33  18  49   0   0   0|   0     0 |  60B  326B|   0     0 |1469   909
 32  23  40   0   0   5|   0   225M| 120B  658B|   0     0 |1788   979
 32  24  35   0   0   8|   0   371M|  60B  150B|   0     0 |1989   932
 32  26  34   0   0   8|   0   377M|  60B  150B|   0     0 |2090   878
 32  29  28   0   0  11|   0   469M| 120B  444B|   0     0 |2389   864
 33  25  33   0   0   9|   0   418M|  60B  134B|   0     0 |2198   875
 26  28  39   0   0   7|   0   220M|  60B  134B|   0     0 |1866   917
 27  26  45   0   0   2|   0    78M| 330B  782B|   0     0 |1672   928
 31  19  50   0   0   0|   0     0 |  60B  150B|   0     0 |1466   908
 31  20  49   0   0   0|   0     0 |  60B  134B|   0     0 |1486   943
 31  19  49   0   0   1|   0     0 | 330B  728B|   0     0 |1484   920
 19  28  53   0   0   0|   0     0 | 924B  785B|   0     0 |1442   879
  2   2  96   0   0   0|   0     0 | 617B  884B|   0     0 | 237   238
 33  18  49   0   0   0|   0     0 | 180B  744B|   0     0 |1568   976
 33  21  43   0   0   3|   0   143M|  60B  118B|   0     0 |1764   970
```

通过iostat可以明显的印证当前系统有很大的磁盘写入：

```
# iostat -x -d 1
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00  655.45     0.00 307536.63   938.40     0.58    0.89    0.00    0.89   0.39  25.35

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00  880.00     0.00 418984.00   952.24     0.78    0.90    0.00    0.90   0.38  33.40

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00 1425.74     0.00 676994.06   949.67     0.98    0.69    0.00    0.69   0.42  59.60

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00  129.00     0.00 61524.00   953.86     0.09    0.71    0.00    0.71   0.43   5.50

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00  989.11     0.00 468352.48   947.02     0.89    0.91    0.00    0.91   0.38  37.23

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00  551.00     0.00 260312.00   944.87     0.35    0.65    0.00    0.65   0.40  21.90

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

通过pidstat找到进程号 10071 的程序在大量写入数据：

```
# pidstat -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/25/2021      _x86_64_        (2 CPU)
10:16:15 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:16:16 PM     0     10071      0.00 415136.63      0.00  python

10:16:16 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:16:17 PM     0     10071      0.00 401548.00      0.00  python

10:16:17 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:16:18 PM     0     10071      0.00 160596.00      0.00  python

10:16:18 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

10:16:19 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
```

通过strace命令进行分析 10071 进程的详细信息，可以发现其子进程 10166 正在打开文件并写入：

```
# strace -p 10071 -f
...
[pid 10166] open("/tmp/1f882e66-bd64-11eb-bbb7-0242ac145802/666.txt", O_RDONLY|O_CLOEXEC) = 6
[pid 10166] fcntl(6, F_SETFD, FD_CLOEXEC) = 0
[pid 10166] fstat(6, {st_mode=S_IFREG|0644, st_size=2599999, ...}) = 0
[pid 10166] ioctl(6, TIOCGWINSZ, 0x7fd4e43f6290) = -1 ENOTTY (Inappropriate ioctl for device)
[pid 10166] lseek(6, 0, SEEK_CUR)       = 0
[pid 10166] ioctl(6, TIOCGWINSZ, 0x7fd4e43f61b0) = -1 ENOTTY (Inappropriate ioctl for device)
[pid 10166] lseek(6, 0, SEEK_CUR)       = 0
[pid 10166] fstat(6, {st_mode=S_IFREG|0644, st_size=2599999, ...}) = 0
[pid 10166] mmap(NULL, 2600960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd4e3b2c000
[pid 10166] read(6, "The king eats a dude to know mor"..., 2600000) = 2599999
[pid 10166] read(6, "", 1)              = 0
[pid 10166] mmap(NULL, 2600960, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd4e38b1000
[pid 10166] munmap(0x7fd4e3b2c000, 2600960) = 0
[pid 10166] munmap(0x7fd4e38b1000, 2600960) = 0
...
```

可以使用 pstree 查看 10071 进程的子进程数量：

```
# pstree -a -p 10071
python(10071)─┬─{python}(10072)
              └─{python}(10162)
```

通过上面的查询基本上以及定位问题的根源，下面是找开发进行优化代码，可以查看源码找到一个临时优化的方法：

```
# time curl http://192.168.120.88:10000/popular/word
{
  "popularity": 0.0,
  "word": "word"
}

real    0m6.890s
user    0m0.002s
sys     0m0.004s
```

### 案例分析三

运行案例：

```
# git clone https://github.com/feiskyer/linux-perf-examples
# cd linux-perf-examples
# make build
# make run
docker run --name=mysql -itd -p 10000:80 -m 800m feisky/mysql:5.6
5a88f0a0ece185b0b37a330df25d9b99047b1559a826d367fa3b0c1e03a3b017
docker run --name=dataservice -itd --privileged feisky/mysql-dataservice
3d3a4c368861f7387920e397f6a0dd6a8c70e4f599b680f7205a94772dca16e6
docker run --name=app --network=container:mysql -itd feisky/mysql-slow
001bddd5e3c93c929ff639b31a87f1cfd1e0de1f7caafcff61a6bcb83e5d589c
# docker ps -a
CONTAINER ID   IMAGE                      COMMAND                  CREATED              STATUS              PORTS                                               NAMES
001bddd5e3c9   feisky/mysql-slow          "python /app.py"         About a minute ago   Up About a minute                                                       app
3d3a4c368861   feisky/mysql-dataservice   "python /dataservice…"   About a minute ago   Up About a minute                                                       dataservice
5a88f0a0ece1   feisky/mysql:5.6           "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp, 0.0.0.0:10000->80/tcp, :::10000->80/tcp   mysql
```

验证是否启动：

```
# curl http://192.168.120.88:10000
Index Page
```

导入数据：

```
# make init
docker exec -i mysql mysql -uroot -P3306 < tables.sql
curl http://127.0.0.1:10000/db/insert/products/10000
insert 10000 lines
```

查询mysql的数据，可以看到速度一般，可以模拟访问请求，进行排查：

```
# curl http://192.168.120.88:10000/products/geektime
Got data: () in 0.5944323539733887 sec
# while true; do curl http://192.168.120.88:10000/products/geektime; sleep 5; done
```

分析问题第一步看下top，基本上没有明显的资源消耗问题：

```
top - 23:17:21 up  1:06,  2 users,  load average: 0.47, 0.42, 0.34
Tasks: 127 total,   1 running, 126 sleeping,   0 stopped,   0 zombie
%Cpu0  : 10.2 us,  6.9 sy,  0.0 ni, 79.7 id,  0.0 wa,  0.0 hi,  3.3 si,  0.0 st
%Cpu1  :  3.8 us,  8.4 sy,  0.0 ni, 86.7 id,  0.0 wa,  0.0 hi,  1.1 si,  0.0 st
KiB Mem :  7990288 total,  7036992 free,   629908 used,   323388 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7005972 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 11193 root      20   0  385636 228780   5376 S  32.2  2.9   6:47.93 python
 10955 polkitd   20   0  836212  57992   8704 S  14.3  0.7   0:15.89 mysqld
 11081 root      20   0   12924   7164   2516 S   1.7  0.1   0:05.52 python
     9 root      20   0       0      0      0 S   0.7  0.0   0:12.43 rcu_sched
     3 root      20   0       0      0      0 S   0.3  0.0   0:02.03 ksoftirqd/0
```

在dstat中可以发现磁盘的读操作有明显的波动：

```
# dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
  5   3  91   0   0   0|1718k   13M|   0     0 |   0     0 | 436   574
  6   4  89   0   0   1|   0     0 |  60B  326B|   0     0 | 435   488
  6   4  90   0   0   0|   0  4096B| 330B  760B|   0     0 | 415   464
  1   1  98   0   0   0|   0     0 |  60B  150B|   0     0 | 152   264
  9  18  69   0   0   3| 489M   12k|1493B 1544B|   0     0 |1047   871
  6   6  88   0   0   0|8192B 4096B| 330B  760B|   0     0 | 441   487
  2   2  96   0   0   0|   0     0 | 120B  194B|   0     0 | 224   346
  2   2  96   0   0   0|   0     0 |  60B  134B|   0     0 | 245   374
  4   2  94   0   0   0|   0  4096B| 120B  412B|   0     0 | 349   464
  4   3  93   0   0   1|  36M 8192B| 709B  978B|   0     0 | 377   518
  3  12  85   0   0   1| 453M    0 | 936B  807B|   0     0 | 627   519
  2   1  98   0   0   0|4096B 9216B| 330B  680B|   0     0 | 206   353
```

也可以用iostat验证当前机器的磁盘读写信息：

```
# iostat -x -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/25/2021      _x86_64_        (2 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     6.79    7.43   35.93  2452.32 13060.62   715.51     0.12    2.85    1.72    3.09   0.36   1.57

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00  979.00    0.00 500440.00     0.00  1022.35     1.63    1.81    1.81    0.00   0.38  37.20

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    1.00    1.00     4.00     4.00     8.00     0.00    1.50    1.00    2.00   1.50   0.30
```

既然当前机器的磁盘读写忽高忽低，可以通过pidstat查看到底哪个进程消耗的：

```
# pidstat -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/25/2021      _x86_64_        (2 CPU)

11:18:43 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

11:18:44 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

11:18:45 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
11:18:46 PM   999     10955 504724.00      0.00      0.00  mysqld
11:18:46 PM     0     11081      4.00      4.00      0.00  python

11:18:46 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

11:18:47 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

11:18:48 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
11:18:49 PM     0     11081      4.00      4.00      0.00  python

11:18:49 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command

11:18:50 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
11:18:51 PM   999     10955 345088.00      0.00      0.00  mysqld

11:18:51 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
11:18:52 PM   999     10955 155344.00      0.00      0.00  mysqld
11:18:52 PM     0     11081      4.00      4.00      0.00  python
```

发现mysql进程的读操作很大可以用strace查看进行的详细信息：

```
# strace -f -p 10955
...
[pid 11328] read(37, "UONXFJ9r286rKk9i3krScZXEOZ71YDDR"..., 20480) = 20480
[pid 11328] read(37, "EZ8tBLLL2QtrP6aS0OCng2XiElCaUwyS"..., 131072) = 131072
[pid 11328] read(37, "517KyAHzK4oYb9EuX6F4ffRwe6SajUcO"..., 24576) = 24576
[pid 11328] read(37, "l0HttIkF6lM17wLMCCxLIdwl4Gr26oOD"..., 131072) = 131072
[pid 11328] read(37, "f99RKWTTpS1oKajYJJnjC0vArhU1JrrW"..., 20480) = 20480
[pid 11328] read(37, "ctJm0391YhgknlC1uGBP0Yl1rQjKB9B8"..., 131072) = 131072
[pid 11328] read(37, "5vq0F4ZPptfsnl8mR9WQ2bUXazSE0c2y"..., 24576) = 24576
[pid 11328] read(37, "LHoKtFZV6p9L74NZ80ypCYje5euvtKZj"..., 131072) = 131072
[pid 11328] read(37, "kh7pntQvl65JvvLW3EDASknzKZR6tdgx"..., 20480) = 20480
[pid 11328] read(37, "xMcx1n6CjXEIQ9Uxt2siCDZawniGklcl"..., 131072) = 131072
```

既然怀疑到MySQL就需登录到MySQL中查找问题：

```
# docker exec -i -t mysql mysql
mysql> show full processlist;
+-----+------+-----------------+------+---------+------+--------------+-----------------------------------------------------+
| Id  | User | Host            | db   | Command | Time | State        | Info                                                |
+-----+------+-----------------+------+---------+------+--------------+-----------------------------------------------------+
| 101 | root | localhost       | NULL | Query   |    0 | init         | show full processlist                               |
| 106 | root | 127.0.0.1:43376 | test | Query   |    0 | Sending data | select * from products where productName='geektime' |
+-----+------+-----------------+------+---------+------+--------------+-----------------------------------------------------+
2 rows in set (0.00 sec)
```

多执行几次 show full processlist 命令，可以看到 select * from products where productName=‘geektime’ 这条 SQL 语句的执行时间比较长，下面查看这条语句的消耗：

```
mysql> use test

mysql> explain select * from products where productName='geektime';
+----+-------------+----------+------+---------------+------+---------+------+-------+-------------+
| id | select_type | table    | type | possible_keys | key  | key_len | ref  | rows  | Extra       |
+----+-------------+----------+------+---------------+------+---------+------+-------+-------------+
|  1 | SIMPLE      | products | ALL  | NULL          | NULL | NULL    | NULL | 10000 | Using where |
+----+-------------+----------+------+---------------+------+---------+------+-------+-------------+
1 row in set (0.02 sec)
```

* select_type 表示查询类型，而这里的 SIMPLE 表示此查询不包括 UNION 查询或者子查询；
* table 表示数据表的名字，这里是 products；
* type 表示查询类型，这里的 ALL 表示全表查询，但索引查询应该是 index 类型才对；
* possible_keys 表示可能选用的索引，这里是 NULL；
* key 表示确切会使用的索引，这里也是 NULL；
* rows 表示查询扫描的行数，这里是 10000。 

 根据这些信息，我们可以确定，这条查询语句压根儿没有使用索引，所以查询时，会扫描全表。下面 查询 products 表的结构 ：

```
mysql> show create table products\G
*************************** 1. row ***************************
       Table: products
Create Table: CREATE TABLE `products` (
  `id` int(11) NOT NULL,
  `productCode` text NOT NULL COMMENT '产品代码',
  `productName` text NOT NULL COMMENT '产品名称',
  `productLine` text NOT NULL COMMENT '产品线',
  `productScale` text NOT NULL,
  `productVendor` text NOT NULL,
  `productDescription` text NOT NULL,
  `quantityInStock` smallint(6) NOT NULL COMMENT '库存',
  `buyPrice` decimal(10,2) NOT NULL COMMENT '价格',
  `MSRP` decimal(10,2) NOT NULL COMMENT '建议零售价',
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
1 row in set (0.00 sec)
```

通过上面可以看到products表只有一个 id 主键 ，下面给  productName  字段创建索引：

```
mysql> CREATE INDEX products_index ON products (productName(64));
Query OK, 10000 rows affected (1.59 sec)
Records: 10000  Duplicates: 0  Warnings: 0
```

在创建完索引后在观察请求响应时间，可以明显的发现速度提升：

```
Got data: () in 0.4852008819580078 sec
Got data: () in 0.4056692123413086 sec
Got data: () in 0.5237679481506348 sec
Got data: () in 0.020385026931762695 sec
Got data: () in 0.0022945404052734375 sec
Got data: () in 0.004728794097900391 sec
Got data: () in 0.00191497802734375 sec
```

### 案例分析四

运行测试案例：

```
# docker run --name=redis -itd -p 10000:80 feisky/redis-server
# docker run --name=app --network=container:redis -itd feisky/redis-app
```

查看测试案例是否启动成功：

```
# curl http://192.168.120.88:10000
hello redis
```

 初始化 Redis 缓存，并且插入 5000 条缓存信息 ：

```
# curl http://192.168.120.88:10000/init/5000
{"elapsed_seconds":3.914144277572632,"keys_initialized":5000}
```

查看运行时间：

```
# time curl http://192.168.120.88:10000/get_cache
{"count":1717,"data":["6b002ff6-c06b-11eb-a142-0242ac145802",..."6a853cce-c06b-11eb-a142-0242ac145802"],"elapsed_seconds":2.681152820587158,"type":"good"}

real    0m2.660s
user    0m0.002s
sys     0m0.004s
```

模拟用户请求

```
# while true; do curl http://192.168.120.88:10000/get_cache;sleep 1; done
```

top命令观察系统第一步：

```
top - 18:54:36 up  1:35,  2 users,  load average: 0.39, 0.12, 0.08
Tasks: 125 total,   1 running, 124 sleeping,   0 stopped,   0 zombie
%Cpu0  :  5.3 us,  3.3 sy,  0.0 ni, 90.8 id,  0.7 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  3.4 us,  2.1 sy,  0.0 ni, 81.5 id, 12.9 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  7990288 total,  5958528 free,   414520 used,  1617240 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  7164164 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 11647 100       20   0   28340   9100   1136 D  48.5  0.1   0:34.37 redis-server
 11722 root      20   0  113020  26492   5292 S  47.8  0.3   0:32.17 python
 10465 root      20   0       0      0      0 S   3.3  0.0   0:06.06 kworker/0:29
  4252 root       0 -20       0      0      0 S   3.0  0.0   0:02.34 kworker/0:1H
 10490 root      20   0       0      0      0 S   1.3  0.0   0:01.07 kworker/1:1
  8838 root      20   0   90392   3212   2348 S   1.0  0.0   0:03.02 rngd
```

通过top命令观察除了发现 wa 稍微搞点，未发现其它问题，下面使用dstat命令观察系统状态：

```
# dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
  0   0  99   0   0   0|  84k  479k|   0     0 |   0     0 | 305   565
  5   4  83   8   0   1|   0  2858k| 134k  102k|   0     0 |8166    16k
  5   4  83   8   0   0|   0  3069k|  60B  166B|   0     0 |8477    17k
  4   2  86   8   0   0|   0  2957k| 133k  102k|   0     0 |8409    17k
  3   4  85   8   0   0|   0  2886k| 330B  798B|   0     0 |8100    16k
  2   2  87   8   0   1|   0  3115k|  60B  134B|   0     0 |8501    17k
  2   3  86   8   0   1|   0  2957k| 133k  102k|   0     0 |8379    17k
  5   5  82   8   0   0|   0  3055k| 120B  530B|   0     0 |8513    17k
  5   5  81   8   0   2|   0  3089k|  60B  134B|   0     0 |8465    17k

```

通过dstat命令能观测到当前系统有个持续写入的动作，并且系统的上下文切换有点高，但是系统的CPU使用率不高，只能怀疑系统的磁盘，下面使用iostat查看当前系统的io状况：

```
# iostat -x -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/29/2021      _x86_64_        (2 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.01     8.75    2.15   28.99    84.03   491.03    36.93     0.04    1.30    0.80    1.34   0.14   0.42

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    1.00 1208.00     8.00  3060.50     5.08     0.12    0.10    1.00    0.10   0.10  12.10

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00 1146.00     0.00  2901.50     5.06     0.13    0.11    0.00    0.11   0.11  12.80

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00 1231.00     0.00  3116.00     5.06     0.10    0.09    0.00    0.09   0.09  10.50

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00 1212.00     0.00  3066.00     5.06     0.10    0.09    0.00    0.09   0.08  10.10

```

通过iostat观察到当前系统是在做数据写入操作，但是并未达到磁盘写入的瓶颈。我们模拟的请求是读操作，而当前系统确实写操作居多，这个有点违背常理，下面使用pidstat查看当前系统使用磁盘的进程信息：

```
# pidstat -d 1
Linux 3.10.0-957.el7.x86_64 (localhost.localdomain)     05/29/2021      _x86_64_        (2 CPU)

06:55:50 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
06:55:51 PM   100     11647      0.00   2340.59      0.00  redis-server

06:55:51 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
06:55:52 PM   100     11647      0.00   2452.00      0.00  redis-server

06:55:52 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
06:55:53 PM   100     11647      0.00   2456.00      0.00  redis-server

06:55:53 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
06:55:54 PM   100     11647      0.00   2316.00      0.00  redis-server
```

当前系统只有redis进程在做磁盘的写入操作，可以使用 strace+lsof 组合，看看 redis-server 到底在写什么 ：

```
# strace -f -T -tt -p 11647
...
[pid 11647] 18:56:32.648675 fdatasync(7) = 0 <0.000993>
[pid 11647] 18:56:32.649861 write(8, ":1\r\n", 4) = 4 <0.000347>
[pid 11647] 18:56:32.650437 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 62, NULL, 8) = 1 <0.000236>
[pid 11647] 18:56:32.650871 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:694fc004-"..., 16384) = 61 <0.000174>
[pid 11647] 18:56:32.651260 read(3, 0x7ffe6df26427, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000218>
[pid 11647] 18:56:32.651776 write(8, "$6\r\nnormal\r\n", 12) = 12 <0.000633>
[pid 11647] 18:56:32.652893 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 60, NULL, 8) = 1 <0.000150>
[pid 11647] 18:56:32.653179 read(8, "*4\r\n$4\r\nSCAN\r\n$4\r\n3847\r\n$5\r\nMATC"..., 16384) = 47 <0.000175>
[pid 11647] 18:56:32.653576 read(3, 0x7ffe6df26427, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000194>
[pid 11647] 18:56:32.653974 write(8, "*2\r\n$4\r\n2695\r\n*10\r\n$41\r\nuuid:697"..., 499) = 499 <0.000423>
[pid 11647] 18:56:32.654679 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 58, NULL, 8) = 1 <0.000140>
[pid 11647] 18:56:32.655020 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:6979c21e-"..., 16384) = 61 <0.000174>
[pid 11647] 18:56:32.655409 read(3, 0x7ffe6df26427, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000174>
[pid 11647] 18:56:32.656073 write(8, "$6\r\nnormal\r\n", 12) = 12 <0.000310>
[pid 11647] 18:56:32.656591 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 56, NULL, 8) = 1 <0.000093>
[pid 11647] 18:56:32.656846 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:699f847c-"..., 16384) = 61 <0.000182>
[pid 11647] 18:56:32.657233 read(3, 0x7ffe6df26427, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000188>
[pid 11647] 18:56:32.657690 write(8, "$4\r\ngood\r\n", 10) = 10 <0.000414>
[pid 11647] 18:56:32.658364 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 54, NULL, 8) = 1 <0.000185>
[pid 11647] 18:56:32.658748 read(8, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n699"..., 16384) = 67 <0.000426>
[pid 11647] 18:56:32.659402 read(3, 0x7ffe6df26427, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000264>
[pid 11647] 18:56:32.659854 write(7, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n699"..., 67) = 67 <0.000190>
[pid 11647] 18:56:32.660278 fdatasync(7) = 0 <0.001054>
...
```

从系统调用来看， epoll_pwait、read、write、fdatasync 这些系统调用都比较频繁。那么，刚才观察到的写磁盘，应该就是 write 或者 fdatasync 导致的了。

 运行 lsof 命令，找出这些系统调用的操作对象： 

````
# lsof -p 11647
...
COMMAND     PID     USER   FD      TYPE DEVICE SIZE/OFF     NODE NAME
...
redis-ser 11647      100  txt       REG   0,38  8191592 53609228 /usr/local/bin/redis-server
...
redis-ser 11647      100    6u     sock    0,7      0t0    70779 protocol: TCP
lsof: no pwd entry for UID 100
redis-ser 11647      100    7w      REG    8,3  9034658  1653995 /data/appendonly.aof
lsof: no pwd entry for UID 100
redis-ser 11647      100    8u     sock    0,7      0t0    71318 protocol: TCP
````

通过上面的数据可以找到redis在操作/data/appendonly.aof文件，运行下面的命令，查询 appendonly 和 appendfsync 的配置

```
# docker exec -it redis redis-cli config get 'append*'
1) "appendfsync"
2) "always"
3) "appendonly"
4) "yes"
```

 appendfsync  是用来设置 fsync 的策略配置选项如下：

* always 表示，每个操作都会执行一次 fsync，是最为安全的方式；

* everysec 表示，每秒钟调用一次 fsync ，这样可以保证即使是最坏情况下，也只丢失 1 秒的数据；

* no 表示交给操作系统来处理。 

 可以用 strace 观察  fdatasync 系统调用的执行情况。比如通过 -e 选项指定 fdatasync 后，你就会得到下面的结果 ：

```
# strace -f -p 11647 -T -tt -e fdatasync
strace: Process 11647 attached with 4 threads
[pid 11647] 18:58:57.438566 fdatasync(7) = 0 <0.000897>
[pid 11647] 18:58:57.442526 fdatasync(7) = 0 <0.000885>
[pid 11647] 18:58:57.465135 fdatasync(7) = 0 <0.000817>
[pid 11647] 18:58:57.468993 fdatasync(7) = 0 <0.000817>
[pid 11647] 18:58:57.477399 fdatasync(7) = 0 <0.000881>
[pid 11647] 18:58:57.486139 fdatasync(7) = 0 <0.000826>
[pid 11647] 18:58:57.496163 fdatasync(7) = 0 <0.000757>
[pid 11647] 18:58:57.511862 fdatasync(7) = 0 <0.000821>
[pid 11647] 18:58:57.517263 fdatasync(7) = 0 <0.000847>
```

 从这里你可以看到，每隔 10ms 左右，就会有一次 fdatasync 调用，并且每次调用本身也要消耗 7~8ms。 

 为什么查询时会有磁盘写呢？ 

根据 lsof 的分析，文件描述符编号为 7 的是一个普通文件 /data/appendonly.aof，而编号为 8 的是 TCP socket。而观察上面的内容，8 号对应的 TCP 读写，是一个标准的“请求 - 响应”格式，即： 

* 从 socket 读取 GET uuid:699f847c-… 后，响应 good；
* 再从 socket 读取 SADD good 699… 后，响应 1。 

对 Redis 来说，SADD 是一个写操作，所以 Redis 还会把它保存到用于持久化的 appendonly.aof 文件中。  观察更多的 strace 结果，你会发现，每当 GET 返回 good 时，随后都会有一个 SADD 操作，这也就导致了，明明是查询接口，Redis 却有大量的磁盘写。 

由于程序运行在容器中可以通过 nsenter 查看容器中的信息：

```
# docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
# PID=$(docker inspect --format {{.State.Pid}} app)
# nsenter --target $PID --net -- lsof -i
lsof: no pwd entry for UID 100
lsof: no pwd entry for UID 100
COMMAND     PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
lsof: no pwd entry for UID 100
redis-ser 11647      100    6u  IPv4  70779      0t0  TCP localhost:6379 (LISTEN)
lsof: no pwd entry for UID 100
redis-ser 11647      100    8u  IPv4  71318      0t0  TCP localhost:6379->localhost:56400 (ESTABLISHED)
python    11722     root    3u  IPv4  72321      0t0  TCP *:http (LISTEN)
python    11722     root    4u  IPv4  81583      0t0  TCP localhost.localdomain:http->192.168.120.74:42804 (ESTABLISHED)
python    11722     root    5u  IPv4  72442      0t0  TCP localhost:56400->localhost:6379 (ESTABLISHED)
```

 这次我们可以看到，redis-server 的 8 号文件描述符，对应 TCP 连接 localhost:6379->localhost:56400。其中， localhost:6379 是 redis-server 自己的监听端口，自然 localhost:56400就是 redis 的客户端。再观察最后一行，localhost:56400对应的，正是我们的 Python 应用程序（进程号为 11722）。 

```
# ps aux |grep 11722
root      11722  3.3  0.3 113020 26604 pts/0    Ss+  18:47   4:34 python /app.py
```

总结一下，我们找到两个问题：

* 第一个问题，Redis 配置的 appendfsync 是 always，这就导致 Redis 每次的写操作，都会触发 fdatasync 系统调用。今天的案例，没必要用这么高频的同步写，使用默认的 1s 时间间隔，就足够了。

* 第二个问题，Python 应用在查询接口中会调用 Redis 的 SADD 命令，这很可能是不合理使用缓存导致的。 

 对于第一个配置问题，我们可以执行下面的命令，把 appendfsync 改成 everysec： 

```
# docker exec -it redis redis-cli config set appendfsync everysec
OK
```

 第二个问题，就要查看应用的源码了。 下面是解决第一个问题后接口请求响应时间：

```
# time curl http://192.168.120.88:10000/get_cache
...
real    0m1.710s
user    0m0.000s
sys     0m0.007s
```

## 优化思路

### I/O 基准测试

为了更客观合理地评估优化效果，我们首先应该对磁盘和文件系统进行基准测试，得到文件系统或者磁盘 I/O 的极限性能。fio  正是最常用的文件系统和磁盘 I/O 性能基准测试工具。它提供了大量的可定制化选项，可以用来测试，裸盘或者文件系统在各种场景下的 I/O 性能，包括了不同块大小、不同 I/O 引擎以及是否使用缓存等场景。 

```
# 随机读
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 随机写
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序读
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序写
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb 
```

* direct，表示是否跳过系统缓存。上面示例中，我设置的 1 ，就表示跳过系统缓存。
* iodepth，表示使用异步 I/O（asynchronous I/O，简称 AIO）时，同时发出的 I/O 请求上限。在上面的示例中，我设置的是 64。
* rw，表示 I/O 模式。我的示例中， read/write 分别表示顺序读 / 写，而 randread/randwrite 则分别表示随机读 / 写。
* ioengine，表示 I/O 引擎，它支持同步（sync）、异步（libaio）、内存映射（mmap）、网络（net）等各种 I/O 引擎。上面示例中，我设置的 libaio 表示使用异步 I/O。
* bs，表示 I/O 的大小。示例中，我设置成了 4K（这也是默认值）。
* filename，表示文件路径，当然，它可以是磁盘路径（测试磁盘性能），也可以是文件路径（测试文件系统性能）。示例中，我把它设置成了磁盘 /dev/sdb。不过注意，用磁盘路径测试写，会破坏这个磁盘中的文件系统，所以在使用前，你一定要事先做好数据备份。 

```
# fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sda
read: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=64
fio-3.7
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=45.9MiB/s,w=0KiB/s][r=11.7k,w=0 IOPS][eta 00m:00s]
read: (groupid=0, jobs=1): err= 0: pid=12698: Sat May 29 21:39:20 2021
   read: IOPS=11.4k, BW=44.6MiB/s (46.8MB/s)(1024MiB/22964msec)
    slat (nsec): min=1161, max=1509.7k, avg=83188.73, stdev=40849.26
    clat (usec): min=2, max=19311, avg=5520.91, stdev=1696.11
     lat (usec): min=91, max=19312, avg=5604.54, stdev=1723.52
    clat percentiles (usec):
     |  1.00th=[ 1254],  5.00th=[ 1778], 10.00th=[ 2147], 20.00th=[ 5014],
     | 30.00th=[ 5932], 40.00th=[ 6128], 50.00th=[ 6194], 60.00th=[ 6390],
     | 70.00th=[ 6390], 80.00th=[ 6456], 90.00th=[ 6587], 95.00th=[ 6652],
     | 99.00th=[ 7570], 99.50th=[ 8717], 99.90th=[10683], 99.95th=[11994],
     | 99.99th=[14877]
   bw (  KiB/s): min=37285, max=109203, per=99.90%, avg=45615.22, stdev=17303.02, samples=45
   iops        : min= 9321, max=27300, avg=11403.58, stdev=4325.72, samples=45
  lat (usec)   : 4=0.01%, 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%
  lat (usec)   : 1000=0.06%
  lat (msec)   : 2=8.14%, 4=11.22%, 10=80.42%, 20=0.17%
  cpu          : usr=3.32%, sys=95.52%, ctx=3763, majf=0, minf=98
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued rwts: total=262144,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: bw=44.6MiB/s (46.8MB/s), 44.6MiB/s-44.6MiB/s (46.8MB/s-46.8MB/s), io=1024MiB (1074MB), run=22964-22964msec

Disk stats (read/write):
  sda: ios=244920/36, merge=16345/11, ticks=59976/17, in_queue=59936, util=92.75%
```

* slat ，是指从 I/O 提交到实际执行 I/O 的时长（Submission latency）；
* clat ，是指从 I/O 提交到 I/O 完成的时长（Completion latency）；
* lat ，指的是从 fio 创建 I/O 到 I/O 完成的总时长。 
* bw ，它代表吞吐量。 

通常情况下，应用程序的 I/O 都是读写并行的，而且每次的 I/O 大小也不一定相同。所以，刚刚说的这几种场景，并不能精确模拟应用程序的 I/O 模式。 fio 支持 I/O 的重放。借助前面提到过的 blktrace，再配合上 fio，就可以实现对应用程序 I/O 模式的基准测试。你需要先用 blktrace ，记录磁盘设备的 I/O 访问情况；然后使用 fio ，重放 blktrace 的记录。 

```

# 使用blktrace跟踪磁盘I/O，注意指定应用程序正在操作的磁盘
$ blktrace /dev/sdb

# 查看blktrace记录的结果
# ls
sdb.blktrace.0  sdb.blktrace.1

# 将结果转化为二进制文件
$ blkparse sdb -d sdb.bin

# 使用fio重放日志
$ fio --name=replay --filename=/dev/sdb --direct=1 --read_iolog=sdb.bin 
```

###  应用程序优化

* 第一，可以用追加写代替随机写，减少寻址开销，加快 I/O 写的速度。
* 第二，可以借助缓存 I/O ，充分利用系统缓存，降低实际 I/O 的次数。
* 第三，可以在应用程序内部构建自己的缓存，或者用 Redis 这类外部缓存系统。这样，一方面，能在应用程序内部，控制缓存的数据和生命周期；另一方面，也能降低其他应用程序使用缓存对自身的影响。 

* 第四，在需要频繁读写同一块磁盘空间时，可以用 mmap 代替 read/write，减少内存的拷贝次数。 
* 第五，在需要同步写的场景中，尽量将写请求合并，而不是让每个请求都同步写入磁盘，即可以用 fsync() 取代 O_SYNC。 

* 第六，在多个应用程序共享相同磁盘时，为了保证 I/O 不被某个应用完全占用，推荐你使用 cgroups 的 I/O 子系统，来限制进程 / 进程组的 IOPS 以及吞吐量。 

*  最后，在使用 CFQ 调度器时，可以用 ionice 来调整进程的 I/O 调度优先级，特别是提高核心应用的 I/O 优先级。ionice 支持三个优先级类：Idle、Best-effort 和 Realtime。其中， Best-effort 和 Realtime 还分别支持 0-7 的级别，数值越小，则表示优先级别越高。 

###  文件系统优化 

*  第一，你可以根据实际负载场景的不同，选择最适合的文件系统。  相比于 ext4 ，xfs 支持更大的磁盘分区和更大的文件数量，如 xfs 支持大于 16TB 的磁盘。但是 xfs 文件系统的缺点在于无法收缩，而 ext4 则可以。 
*  第二，在选好文件系统后，还可以进一步优化文件系统的配置选项，包括文件系统的特性（如 ext_attr、dir_index）、日志模式（如 journal、ordered、writeback）、挂载选项（如 noatime）等等。  比如， 使用 tune2fs 这个工具，可以调整文件系统的特性（tune2fs 也常用来查看文件系统超级块的内容）。 
* 第三，可以优化文件系统的缓存。  比如，你可以优化 pdflush 脏页的刷新频率（比如设置 dirty_expire_centisecs 和 dirty_writeback_centisecs）以及脏页的限额（比如调整 dirty_background_ratio 和 dirty_ratio 等）。  再如，你还可以优化内核回收目录项缓存和索引节点缓存的倾向，即调整 vfs_cache_pressure（/proc/sys/vm/vfs_cache_pressure，默认值 100），数值越大，就表示越容易回收。 
* 最后，在不需要持久化时，你还可以用内存文件系统 tmpfs，以获得更好的 I/O 性能 。tmpfs 把数据直接保存在内存中，而不是磁盘中。比如 /dev/shm/ ，就是大多数 Linux 默认配置的一个内存文件系统，它的大小默认为总内存的一半。 

###  磁盘优化 

* 第一，最简单有效的优化方法，就是换用性能更好的磁盘，比如用 SSD 替代 HDD。
* 第二，我们可以使用 RAID ，把多块磁盘组合成一个逻辑磁盘，构成冗余独立磁盘阵列。这样做既可以提高数据的可靠性，又可以提升数据的访问性能。
* 第三，针对磁盘和应用程序 I/O 模式的特征，我们可以选择最适合的 I/O 调度算法。比方说，SSD 和虚拟机中的磁盘，通常用的是 noop 调度算法。而数据库应用，我更推荐使用 deadline 算法。
* 第四，我们可以对应用程序的数据，进行磁盘级别的隔离。比如，我们可以为日志、数据库等 I/O 压力比较重的应用，配置单独的磁盘。
* 第五，在顺序读比较多的场景中，我们可以增大磁盘的预读数据，比如，你可以通过下面两种方法，调整 /dev/sdb 的预读大小。 
  *  调整内核选项 /sys/block/sdb/queue/read_ahead_kb，默认大小是 128 KB，单位为 KB。 
  *  使用 blockdev 工具设置，比如 blockdev --setra 8192 /dev/sdb，注意这里的单位是 512B（0.5KB），所以它的数值总是 read_ahead_kb 的两倍。 
*  第六，我们可以优化内核块设备 I/O 的选项。比如，可以调整磁盘队列的长度 /sys/block/sdb/queue/nr_requests，适当增大队列长度，可以提升磁盘的吞吐量（当然也会导致 I/O 延迟增大）。 
* 最后，要注意，磁盘本身出现硬件错误，也会导致 I/O 性能急剧下降，所以发现磁盘性能急剧下降时，你还需要确认，磁盘本身是不是出现了硬件错误。  比如，你可以查看 dmesg 中是否有硬件 I/O 故障的日志。 还可以使用 badblocks、smartctl 等工具，检测磁盘的硬件问题，或用 e2fsck 等来检测文件系统的错误。如果发现问题，你可以使用 fsck 等工具来修复。 