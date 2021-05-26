## weave 网络

### 简介

Weave是由weaveworks公司开发的解决Docker跨主机网络的解决方案，它能够创建一个虚拟网络，用于连接部署在多台主机上的Docker容器，这样容器就像被接入了同一个网络交换机，那些使用网络的应用程序不必去配置端口映射和链接等信息。

外部设备能够访问Weave网络上的应用程序容器所提供的服务，同时已有的内部系统也能够暴露到应用程序容器上。Weave能够穿透防火墙并运行在部分连接的网络上，另外，Weave的通信支持加密，所以用户可以从一个不受信任的网络连接到主机。

项目地址：<https://github.com/weaveworks/weave>











## weave 网络环境搭建

### 安装

```shell
[root@host1 ~]# curl -L git.io/weave -o /usr/local/bin/weave
[root@host1 ~]# chmod a+x /usr/local/bin/weave
[root@host1 ~]# weave launch 
[root@host1 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
dfee7f605771        bridge              bridge              local
b4367e71a1ae        host                host                local
65d5164f984f        none                null                local
283192dfea90        weave               weavemesh           local
```

```shell
[root@host2 ~]# curl -L git.io/weave -o /usr/local/bin/weave
[root@host2 ~]# chmod a+x /usr/local/bin/weave
[root@host2 ~]#  weave launch
[root@host2 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
0f2c0996c430        bridge              bridge              local
217ae2b634ca        host                host                local
9244c508018b        none                null                local
ac4584a62c1a        weave               weavemesh           local
```

### 启动容器

host1 主机

```shell
[root@host1 ~]# eval  $(weave env)
[root@host1 ~]# docker run -itd --name busybox_1 busybox
[root@host1 ~]# docker exec busybox_1 ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
12: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:0a:02:28:02 brd ff:ff:ff:ff:ff:ff
    inet 10.2.40.2/24 brd 10.2.40.255 scope global eth0
       valid_lft forever preferred_lft forever
14: ethwe@if15: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1376 qdisc noqueue 
    link/ether 3e:b1:ef:2c:cb:e2 brd ff:ff:ff:ff:ff:ff
    inet 10.32.0.1/12 brd 10.47.255.255 scope global ethwe
       valid_lft forever preferred_lft forever
```



