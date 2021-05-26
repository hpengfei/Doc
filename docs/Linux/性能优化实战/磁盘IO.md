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

