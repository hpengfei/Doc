## 简介

常用基本功能：

* 是扫描主机端口，嗅探所提供的网络服务

* 是探测一组主机是否在线

* 还可以推断主机所用的操作系统，到达主机经过的路由，系统已开放端口的软件版本

 nmap端口状态解析 ：

* open ： 应用程序在该端口接收 TCP 连接或者 UDP 报文。 
* closed ：关闭的端口对于nmap也是可访问的， 它接收nmap探测报文并作出响应。但没有应用程序在其上监听。
* filtered ：由于包过滤阻止探测报文到达端口，nmap无法确定该端口是否开放。过滤可能来自专业的防火墙设备，路由规则 或者主机上的软件防火墙。
* unfiltered ：未被过滤状态意味着端口可访问，但是nmap无法确定它是开放还是关闭。 只有用于映射防火墙规则集的 ACK 扫描才会把端口分类到这个状态。
* open | filtered ：无法确定端口是开放还是被过滤， 开放的端口不响应就是一个例子。没有响应也可能意味着报文过滤器丢弃了探测报文或者它引发的任何反应。UDP，IP协议,FIN, Null 等扫描会引起。
* closed|filtered：（关闭或者被过滤的）：无法确定端口是关闭的还是被过滤的

## 安装

```shell
# yum install -y nmap
```

[官网地址]( https://nmap.org/ )

## 参数介绍

### TARGET SPECIFICATION

```
-iL filename                    从文件中读取待检测的目标,文件中的表示方法支持机名,ip,网段
-iR hostnum                     随机选取,进行扫描.如果-iR指定为0,则是无休止的扫描
--exclude host1[, host2]        从扫描任务中需要排除的主机           
--exculdefile exclude_file      排除文件中的IP,格式和-iL指定扫描文件的格式相同
```

### HOST DISCOVERY

```
-sL                     仅仅是显示,扫描的IP数目,不会进行任何扫描
-sn                     ping扫描,即主机发现
-Pn                     不检测主机存活
-PS/PA/PU/PY[portlist]  TCP SYN Ping/TCP ACK Ping/UDP Ping发现
-PE/PP/PM               使用ICMP echo, timestamp and netmask 请求包发现主机
-PO[prococol list]      使用IP协议包探测对方主机是否开启   
-n/-R                   不对IP进行域名反向解析/为所有的IP都进行域名的反响解析
```

### SCAN TECHNIQUES

```
-sS/sT/sA/sW/sM                 TCP SYN/TCP connect()/ACK/TCP窗口扫描/TCP Maimon扫描
-sU                             UDP扫描
-sN/sF/sX                       TCP NullFINand Xmas扫描
--scanflags                     自定义TCP包中的flags
-sI zombie host[:probeport]     Idlescan
-sY/sZ                          SCTP INIT/COOKIE-ECHO 扫描
-sO                             使用IP protocol 扫描确定目标机支持的协议类型
-b “FTP relay host”             使用FTP bounce scan
```

### PORT SPECIFICATION AND SCAN ORDER

```
-p                      特定的端口 -p80,443 或者 -p1-65535
-p U:PORT               扫描udp的某个端口, -p U:53
-F                      快速扫描模式,比默认的扫描端口还少
-r                      不随机扫描端口,默认是随机扫描的
--top-ports "number"    扫描开放概率最高的number个端口,出现的概率需要参考nmap-services文件,ubuntu中该文件位于/usr/share/nmap.nmap默认扫前1000个
--port-ratio "ratio"    扫描指定频率以上的端口
```

### SERVICE/VERSION DETECTION

```
-sV                             开放版本探测,可以直接使用-A同时打开操作系统探测和版本探测
--version-intensity "level"     设置版本扫描强度,强度水平说明了应该使用哪些探测报文。数值越高服务越有可能被正确识别。默认是7
--version-light                 打开轻量级模式,为--version-intensity 2的别名
--version-all                   尝试所有探测,为--version-intensity 9的别名
--version-trace                 显示出详细的版本侦测过程信息
```

### SCRIPT SCAN

```
-sC                             根据端口识别的服务,调用默认脚本
--script=”Lua scripts”          调用的脚本名
--script-args=n1=v1,[n2=v2]     调用的脚本传递的参数
--script-args-file=filename     使用文本传递参数
--script-trace                  显示所有发送和接收到的数据
--script-updatedb               更新脚本的数据库
--script-help=”Lua script”      显示指定脚本的帮助
```

### OS DETECTION

```
-O              启用操作系统检测,-A来同时启用操作系统检测和版本检测
--osscan-limit  针对指定的目标进行操作系统检测(至少需确知该主机分别有一个open和closed的端口)
--osscan-guess  推测操作系统检测结果,当Nmap无法确定所检测的操作系统时会尽可能地提供最相近的匹配Nmap默认进行这种匹配
```

### FIREWALL/IDS EVASION AND SPOOFING

```
-f; --mtu value                 指定使用分片、指定数据包的MTU.
-D decoy1,decoy2,ME             使用诱饵隐蔽扫描
-S IP-ADDRESS                   源地址欺骗
-e interface                    使用指定的接口
-g/ --source-port PROTNUM       使用指定源端口  
--proxies url1,[url2],...       使用HTTP或者SOCKS4的代理 

--data-length NUM               填充随机数据让数据包长度达到NUM
--ip-options OPTIONS            使用指定的IP选项来发送数据包
--ttl VALUE                     设置IP time-to-live域
--spoof-mac ADDR/PREFIX/VEBDOR  MAC地址伪装
--badsum                        使用错误的checksum来发送数据包
```

### OUTPUT

```
-oN                     将标准输出直接写入指定的文件
-oX                     输出xml文件
-oS                     将所有的输出都改为大写
-oG                     输出便于通过bash或者perl处理的格式,非xml
-oA BASENAME            可将扫描结果以标准格式、XML格式和Grep格式一次性输出
-v                      提高输出信息的详细度
-d level                设置debug级别,最高是9
--reason                显示端口处于带确认状态的原因
--open                  只输出端口状态为open的端口
--packet-trace          显示所有发送或者接收到的数据包
--iflist                显示路由信息和接口,便于调试
--log-errors            把日志等级为errors/warings的日志输出
--append-output         追加到指定的文件
--resume FILENAME       恢复已停止的扫描
--stylesheet PATH/URL   设置XSL样式表转换XML输出
--webxml                从namp.org得到XML的样式
--no-sytlesheet         忽略XML声明的XSL样式表
```

### MISC

```
-6                      开启IPv6
-A                      OS识别,版本探测,脚本扫描和traceroute
--datedir DIRNAME       说明用户Nmap数据文件位置
--send-eth / --send-ip  使用原以太网帧发送/在原IP层发送
--privileged            假定用户具有全部权限
--unprovoleged          假定用户不具有全部权限,创建原始套接字需要root权限
-V                      打印版本信息
-h                      输出帮助
```

## 常用操作

### 快速扫描

 Nmap 默认发送一个arp的 ping 数据包来探测目标主机在1-10000范围内所开放的端口。 

```shell
# nmap 192.168.23.55

Starting Nmap 6.40 ( http://nmap.org ) at 2020-04-28 13:34 CST
Nmap scan report for 192.168.23.55
Host is up (0.0020s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 4.84 seconds
```

### 扫描多个目标

```shell
# nmap -n  192.168.23.55 192.168.24.112

Starting Nmap 6.40 ( http://nmap.org ) at 2020-04-28 14:03 CST
Nmap scan report for 192.168.23.55
Host is up (0.0015s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql

Nmap scan report for 192.168.24.112
Host is up (0.0016s latency).
Not shown: 901 filtered ports, 95 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
8080/tcp open  http-proxy

Nmap done: 2 IP addresses (2 hosts up) scanned in 12.13 seconds
```

### 详细描述输出扫描

```shell
# nmap -n -vv  192.168.23.55 192.168.24.112
```

### 指定端口和范围扫描

```shell
# nmap -p3306,20-80 192.168.23.55   

Starting Nmap 6.40 ( http://nmap.org ) at 2020-04-28 14:06 CST
Nmap scan report for 192.168.23.55
Host is up (0.0018s latency).
Not shown: 59 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 1.69 seconds
```

### 扫描除某一个 IP 外的所有子网主机

```shell
nmap 10.130.1.1/24 -exclude 10.130.1.1,10.130.1.2
```

### 扫描除了某一个文件中 IP 外的子网主机

```shell
nmap 10.130.1.1/24 -excludefile gov.txt
```

### 显示扫描的所有主机列表

```shell
# nmap -sL 192.168.23.1/24
```

### sP ping 扫描

常用来去扫描内网的一个 ip 范围用来做内网的主机发现。 

```shell
# nmap -sP 192.168.23.55-60
```

 PING扫描不同于其它的扫描方式因为它只用于找出主机是否是存在在网络中的.它不是用来发现是否开放端口的.PING扫描需要ROOT权限如果用户没有ROOT权限,PING扫描将会使用connect()调用。

### sS SYN 半开放扫描

```shell
# nmap -sS 192.168.23.55
```

 Tcp SYN Scan (sS) 这是一个基本的扫描方式,它被称为半开放扫描因为这种技术使得Nmap不需要通过完整的握手就能获得远程主机的信息。Nmap发送SYN包到远程主机但是它不会产生任何会话.因此 不会在目标主机上产生任何日志记录,因为没有形成会话。这个就是SYN扫描的优势.如果Nmap命令中没有指出扫描类型,默认的就是`Tcp SYN`.但是它需要`root/administrator`权限。 

### sT TCP 扫描

```shell
# nmap -sT 192.168.23.55
```

 不同于Tcp SYN扫描,Tcp connect()扫描需要完成三次握手,并且要求调用系统的connect().Tcp connect()扫描技术只适用于找出TCP和UDP端口。 

### sU UDP 扫描

```shell
# nmap -sU 192.168.23.55
```

 这种扫描技术用来寻找目标主机打开的UDP端口.它不需要发送任何的SYN包因为这种技术是针对UDP端口的。UDP扫描发送UDP数据包到目标主机并等待响应,如果返回ICMP不可达的错误消息说明端口是关闭的如果得到正确的适当的回应说明端口是开放的. 

### sF FIN 标志的数据包扫描

```shell
# nmap -sF 192.168.23.55
```

### sV Version 版本检测扫描

```shell
#  nmap -sV 192.168.23.55

Starting Nmap 6.40 ( http://nmap.org ) at 2020-04-29 15:32 CST
Nmap scan report for 192.168.23.55
Host is up (0.0023s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http    nginx 1.16.1
3306/tcp open  mysql   MySQL (unauthorized)

Service detection performed. Please report any incorrect results at http://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.36 seconds
```

 本检测是用来扫描目标主机和端口上运行的软件的版本.它不同于其它的扫描技术它不是用来扫描目标主机上开放的端口不过它需要从开放的端口获取信息来判断软件的版本.使用版本检测扫描之前需要先用TCP SYN扫描开放了哪些端口。 

### O OS 操作系统类型的检测

```shell
# nmap -O 192.168.23.55
```

远程检测操作系统和软件Nmap的OS检测技术在渗透测试中用来了解远程主机的操作系统和软件是非常有用的通过获取的信息你可以知道已知的漏洞。Nmap有一个名为的nmap-OS-DB数据库该数据库包含超过2600操作系统的信息。Nmap把TCP和UDP数据包发送到目标机器上然后检查结果和数据库对照。 

### osscan-guess 猜测匹配操作系统

```shell
# nmap -O --osscan-guess 192.168.23.55
```

 通过Nmap准确的检测到远程操作系统是比较困难的需要使用到Nmap的猜测功能选项,`–osscan-guess`猜测认为最接近目标的匹配操作系统类型。 

### PN No ping 扫描

```shell
# nmap -O -PN 192.168.23.1/24
```

 如果远程主机有防火墙IDS和IPS系统你可以使用-PN命令来确保不ping远程主机因为有时候防火墙会组织掉ping请求.-PN命令告诉Nmap不用ping远程主机。使用-PN参数可以绕过PING命令,但是不影响主机的系统的发现。 

### 网段扫描格式

```shell
nmap -sP <network address > </CIDR >
```

 解释CIDR 为你设置的子网掩码`(/24 , /16 ,/8 等)` 

### 从文件中读取需要扫描的 IP 列表

```shell
# cat ip.txt 
192.168.23.55
192.168.24.112
# nmap -iL ip.txt
```

### 路由跟踪扫描

```shell
# nmap -traceroute www.baidu.com

Starting Nmap 6.40 ( http://nmap.org ) at 2020-04-30 11:16 CST
Nmap scan report for www.baidu.com (180.101.49.11)
Host is up (0.010s latency).
Other addresses for www.baidu.com (not scanned): 180.101.49.12
Not shown: 998 filtered ports
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https

TRACEROUTE (using port 80/tcp)
HOP RTT     ADDRESS
1   0.18 ms 192.168.120.2
2   0.20 ms 180.101.49.11

Nmap done: 1 IP address (1 host up) scanned in 52.28 seconds
```

### A OS 识别，版本探测，脚本扫描和traceroute 综合扫描

 此选项设置包含了1-10000的端口ping扫描操作系统扫描脚本扫描路由跟踪服务探测。 

```shell
# nmap -A 192.168.23.55
```

### 命令混合式扫描

命令混合扫描可以做到类似参数-A所完成的功能但又能细化到我们所需特殊要求。

```shell
# nmap -vv -p1-100,3306 -O -traceroute 192.168.23.55
```

 使`SYN`扫描并进行Version版本检测 使用T4(aggressive)的时间模板对目标ip的全端口进行扫描。 

```shell
nmap -p1-65535 -sV -sS -T4 10.130.1.134
```

### 按照脚本进行扫描

```shell
# nmap --script 具体的脚本 www.baidu.com
```

namp 脚本存储位子：

```shell
/usr/share/nmap/scripts/
```

官网使用介绍： https://nmap.org/nsedoc/ 

  



 https://www.freebuf.com/news/141607.html 