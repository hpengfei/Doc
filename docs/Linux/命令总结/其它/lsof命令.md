## 简介

 lsof（list open files）是一个查看当前系统文件的工具。在linux环境下，任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。如传输控制协议 (TCP) 和用户数据报协议 (UDP) 套接字等，系统在后台都为该应用程序分配了一个文件描述符，该文件描述符提供了大量关于这个应用程序本身的信息。 

## 命令参数

- -a 列出打开文件存在的进程
- -c<进程名> 列出指定进程所打开的文件
- -g 列出GID号进程详情
- -d<文件号> 列出占用该文件号的进程
- +d<目录> 列出目录下被打开的文件
- +D<目录> 递归列出目录下被打开的文件
- -n<目录> 列出使用NFS的文件
- -i<条件> 列出符合条件的进程。（4、6、协议、:端口、 @ip ）
- -p<进程号> 列出指定进程号所打开的文件
- -u 列出UID号进程详情

## 使用举例

### 查找某个文件相关进程

```shell
# lsof /bin/bash
COMMAND  PID USER  FD   TYPE DEVICE SIZE/OFF     NODE NAME
bash    7065 root txt    REG  253,0   964608 50332569 /usr/bin/bash
```

### 查找某个用户打开的文件

```shell
# lsof -u username
```

### 列出某个程序打开的文件

```shell
# lsof -c mysql
```

 -c 选项将会列出所有以mysql这个进程开头的程序的文件，其实你也可以写成 lsof | grep mysql, 但是第一种方法明显比第二种方法要少打几个字符； 

### 列出某个用户以及某个进程所打开的文件信息

```shell
# lsof -u username -c command
```

### 通过某个进程号显示该进程打开的文件

```shell
# lsof -p pid
```

### 列出所有的网络连接

```shell
# lsof -ni
```

### 列出所有的 tcp 网络连接信息

```shell
# lsof -ni tcp
```

### 列出谁在使用某个端口

```shell
# lsof -ni :22
```

### 列出某个用户的所有活跃的网络端口

```shell
# lsof -an -u root -i
```

### 列出被进程号为1234的进程所打开的所有IPV4 network files

```shell
# lsof -ni 4 -a -p 1234
```

### 列出目前连接 192.168.23.55 主机上22，80端口相关的文件，每隔3秒输出

```shell
# lsof -ni @192.168.23.55:22,80 -r 3 
COMMAND  PID  USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
sshd    5368  root    3u  IPv4 131055323      0t0  TCP 192.168.23.55:ssh->10.102.25.142:13647 (ESTABLISHED)
nginx   9002 nginx   15u  IPv4 130649886      0t0  TCP 192.168.23.55:http->192.168.103.157:14721 (ESTABLISHED)
nginx   9003 nginx   13u  IPv4 131213598      0t0  TCP 192.168.23.55:http->192.168.103.157:14718 (ESTABLISHED)
```



