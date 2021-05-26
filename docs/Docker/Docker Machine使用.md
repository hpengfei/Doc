## Docker Machine
Docker Machine 是 Docker 官方编排（Orchestration）项目之一，负责在多种平台上快速安装 Docker 环境。
### 说明
|hostname|IP|安装软件|
|:--:|:--:|:--:|
|C7-node3|192.168.120.73|Doker-ce、docker-machine|
|host2|192.168.120.72|Doker-ce|
|host1|192.168.120.71|Doker-ce|

### 安装 `docker-machine`
[下载地址](https://github.com/docker/machine/releases)

On Linux
```shell
curl -L https://github.com/docker/machine/releases/download/v0.16.1/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine
chmod +x /tmp/docker-machine &&
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
# docker-machine -v
docker-machine version 0.16.1, build cce350d7
```
### tab 补全配置
[下载文件1](https://raw.githubusercontent.com/docker/machine/master/contrib/completion/bash/docker-machine.bash)

[下载文件2](https://raw.githubusercontent.com/docker/machine/master/contrib/completion/bash/docker-machine-prompt.bash)

[下载文件3](https://raw.githubusercontent.com/docker/machine/master/contrib/completion/bash/docker-machine-wrapper.bash)

```shell
wget https://raw.githubusercontent.com/docker/machine/master/contrib/completion/bash/docker-machine.bash
wget https://raw.githubusercontent.com/docker/machine/master/contrib/completion/bash/docker-machine-prompt.bash
wget https://raw.githubusercontent.com/docker/machine/master/contrib/completion/bash/docker-machine-wrapper.bash
mv docker-machine* /etc/bash_completion.d/
cd /etc/bash_completion.d/
source docker-machine-prompt.bash
source docker-machine-wrapper.bash
source docker-machine.bash
# tail -n 1 /root/.bashrc  
PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '
# source /root/.bashrc
```
### 创建 Machine
首先要做好 sshd 的公钥私钥的配对：
```shell
# ssh-keygen -t rsa -b 2048
# ssh-copy-id 192.168.120.72
......
root@192.168.120.72's password:
......
# ssh-copy-id 192.168.120.71
```
下面是在 `192.168.120.71` 主机通过 `docker-machine` 进行安装 `docker`：
```shell
# docker-machine create --driver generic --generic-ip-address 192.168.120.71 host1
```
关于参数中设备类型的选择，可以查看官方资料。

[传送门](https://docs.docker.com/machine/drivers/)

在使用上面的安装方式时，你会发现速度很慢，以及不能指定 `docker` 的版本。而通过使用 `--engine-registry-mirror` 指定阿里的docker仓库，以及通过修改官网的 `https://get.docker.com/` 脚本即可快速的安装需要的版本。
```shell
# docker-machine create --driver generic --generic-ip-address 192.168.120.72 --engine-registry-mirror "https://xxxxxx.mirror.aliyuncs.com" --engine-install-url "http://192.168.24.147/install_docker.sh" host2
```
* `install_docker.sh` 脚本[下载地址](https://get.docker.com/)。下载后阅读脚本，修改如下 `pkg_version="-18.09.1"`，修改的位置根据系统的不同而所在的位置不通。
* 创建命令中最后的参数 `host1` 是会修改目标主机的主机名的。 
* docker 的启动参数配置位置： `/etc/systemd/system/docker.service.d/10-machine.conf`。

### 使用 Machine
查看可管理的 docker ：
```shell
# docker-machine ls
NAME    ACTIVE   DRIVER    STATE     URL                         SWARM   DOCKER     ERRORS
host1   -        generic   Running   tcp://192.168.120.71:2376           v18.09.1   
host2   -        generic   Running   tcp://192.168.120.72:2376           v18.09.6   
```
查看管理主机的变量：
```shell
# docker-machine env host1 
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.120.71:2376"
export DOCKER_CERT_PATH="/root/.docker/machine/machines/host1"
export DOCKER_MACHINE_NAME="host1"
# Run this command to configure your shell: 
# eval $(docker-machine env host1)
```
登录到指定主机：
```shell
[root@C7-node3 ~]# eval $(docker-machine env host2)
[root@C7-node3 ~ [host2]]# 
```
增加管理的主机：
```shell
# docker-machine create -d none  --url=tcp://192.168.120.73:2376 host3
```