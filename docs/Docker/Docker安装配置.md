## 安装配置
**说明：这里的操作全部基于 `CentOS 7.4.1708` 学习使用。**
### `yum` 安装
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce-18.09.6
```
```shell
[root@localhost ~]# yum list docker-ce --show-duplicates
```

### 二进制安装

[下载地址](https://download.docker.com/linux/static/stable/x86_64/)
```shell
# wget https://download.docker.com/linux/static/stable/x86_64/docker-18.09.5.tgz
# tar -xf docker-18.09.5.tgz -C /opt/
```
### 添加内核参数
```shell
# modprobe br_netfilter
# tee -a /etc/sysctl.conf <<-EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# sysctl -p
```
### 镜像加速器
建议使用阿里云的镜像加速，地址需要登录阿里云后台进行生成。

[阿里云后台地址](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)

```shell
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn"]
}
EOF
```
### 修改存储路径
```json
{
    "data-root": "/mnt/docker-data",
    "storage-driver": "overlay2"
}
```
### 保持容器在线
```json
{
  "live-restore": true
}
```
### 配置文件

```
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "insecure-registries": ["harbor.k8s.com","quay.io"],
  "registry-mirrors": ["https://2980aqtg.mirror.aliyuncs.com"],
  "bip": "172.20.71.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
```

data-root：Docker运行时使用的根路径，默认/var/lib/docker

storage-driver: 存储驱动，即分层文件系统

insecure-registries: 不安全的docker registries，配置docker的私库地址

registry-mirrors： 加速站点

bip：指定docker bridge地址(不能以.0结尾)，生产中建议采用 172.xx.yy.1/24,其中xx.yy为宿主机ip后四位，方便定位问题

### 启动服务

`yum` 安装的 `docker` 启动方式：
```shell
systemctl enable docker
systemctl start docker
```
二进制的 `docker` 启动方式：
```shell
/opt/docker/dockerd &
```
### 允许远程登录
```shell
# grep tcp /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0 -H fd:// --containerd=/run/containerd/containerd.sock
# systemctl daemon-reload
# systemctl restart docker
```
### 删除 `Docker`
```shell
rpm -e docker-ce
rpm -e docker-ce-cli
rm -rf /var/lib/docker
```