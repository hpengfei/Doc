## sysbench

### 简介

sysbench是跨平台的基准测试工具，支持多线程，支持多种数据库；主要包括以下几种测试：

- cpu性能
- 磁盘io性能
- 调度程序性能
- 内存分配及传输速度
- POSIX线程性能
- 数据库性能(OLTP基准测试)

### 安装

#### 二进制安装

##### Ubuntu

```shell
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```

##### CentOS

```shell
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench
```

#### 源码安装

##### Ubuntu

```shell
# apt -y install make automake libtool pkg-config libaio-dev
#For MySQL support
# apt -y install libmysqlclient-dev libssl-dev
#For PostgreSQL support
# apt -y install libpq-dev
```

##### CentOS

```shell
# yum -y install make automake libtool pkgconfig libaio-devel
#For MySQL support, replace with mysql-devel on RHEL/CentOS 5
# yum -y install mariadb-devel openssl-devel
#For PostgreSQL support
# yum -y install postgresql-devel
```

##### install

```shell
# ./autogen.sh
# ./configure --prefix=/opt/sysbench --with-mysql-includes=/usr/local/mysql/include --with-mysql-libs=/usr/local/mysql/lib
# make -j 2
# make install
# ln -sv /opt/sysbench/bin/sysbench /usr/local/bin/
```

安装所指定的mysql要把其lib库加入到系统配置中，即在`/etc/ld.so.conf.d/` 目录下创建lib库的绝对路径，并通过`ldconfig` 读取。











































<https://github.com/akopytov/sysbench>

<https://www.cnblogs.com/zhoujinyi/archive/2013/04/19/3029134.html>

<https://www.cnblogs.com/kismetv/p/7615738.html>