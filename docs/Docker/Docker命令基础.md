## `Docker` 镜像命令使用
### 下载镜像 `pull`
```shell
# docker pull nginx:1.14.2
# docker pull centos
```
### 查看镜像 `images`
```shell
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               1.14.2              295c7be07902        7 weeks ago         109MB
centos              latest              9f38484d220f        2 months ago        202MB
```
### 修改镜像名 `tag`
```shell
# docker tag centos:latest centos:7.6.1810
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               1.14.2              295c7be07902        7 weeks ago         109MB
centos              7.6.1810            9f38484d220f        2 months ago        202MB
centos              latest              9f38484d220f        2 months ago        202MB
```
### 登录镜像仓库 `login`
docker login 默认登录的是 `https://hub.docker.com/` ，这里使用的是 harbor 搭建的私有仓库的上传镜像。上传之前需要在 `/etc/docker/daemon.json` 文件中增加如下配置：
```shell
{
  "registry-mirrors": ["https://xxx.xxx.com"],
  "insecure-registries": ["192.168.24.112:8080"]
}
```
登录到仓库：
```shell
# docker login 192.168.24.112:8080
Username: example
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```
### 上传镜像 `push`
上传操作如下：
```shell
# docker tag nginx:1.14.2 192.168.24.112:8080/example/nginx:1.14.2
# docker push 192.168.24.112:8080/example/nginx:1.14.2
# docker tag centos:latest 192.168.24.112:8080/example/centos:7.6.1810
# docker push 192.168.24.112:8080/example/centos:7.6.1810
```
### 退出登录 `logout`
```shell
# docker login 192.168.24.112:8080
```
### 删除镜像 `rmi`
```shell
# docker rmi centos:7.6.1810
```
如果一个镜像对应多个 `tag`，只有当最后一个 `tag` 被删除之后才是真正的删除。
### 镜像归档 `save`
```shell
# docker save -o nginx:1.14.2.tar nginx:1.14.2
```
### 镜像导入 `import`
```shell
# docker import nginx\:1.14.2.tar nginx:1.14.2
# docker pull 192.168.24.112:8080/example/nginx:1.14.2
# docker images
REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
nginx                               1.14.2              61e2d997bc2c        3 seconds ago       113MB
192.168.24.112:8080/example/nginx   1.14.2              295c7be07902        8 weeks ago         109MB
```
可以看到使用 import 导入后镜像的 `IMAGE ID` 发生改变，建议镜像上传到远程仓库在进行下载使用。
### 镜像构建历史 `history`
```shell
# docker history nginx:1.14.2
# docker history 192.168.24.112:8080/example/nginx:1.14.2
```
通过构建历史可以发现两个镜像还是有区别的。
### 搜索镜像 `search`
```shell
# docker search xxx
```
## `Docker` 容器命令使用
### 运行容器 `run`
子选项：

* -d: 后台运行容器，并返回容器ID；
* -i: 以交互模式运行容器，通常与 -t 同时使用；
* -t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
* --name="nginx-lb": 为容器指定一个名称；
* -h "mars": 指定容器的hostname；
* -e username="ritchie": 设置环境变量；
* --env-file=[]: 从指定文件读入环境变量；
* --cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；
* -m :设置容器使用内存最大值；
* --net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；
* --link=[]: 添加链接到另一个容器；
* --expose=[]: 开放一个端口或一组端口；
* -p, --publish value： 开启容器时将容器的端口与主机的端口进行绑定；
* -P, --publish-all：将所有暴露的端口发布到随机端口；
* -v, --volume value：绑定挂载的卷；
* --volumes-from value：从指定的容器挂载卷；
* -w, --workdir string：容器中的工作目录
* --restart string：Restart policy to apply when a container exits (default "no")

运行一个测试容器，使用之后自动删除：
```shell
# docker run -it --name container_test --rm busybox:latest sh
```
运行一个容器在后台，通过 `bash` 进行交互操作：
```shell
# docker run -it -d --name centos7.6 centos:7.6.1810 /bin/bash
# docker container ls -l
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
4bc6404d8391        centos:7.6.1810     "/bin/bash"         19 seconds ago      Up 17 seconds                           centos7.6
# docker exec  -it centos7.6  /bin/bash
```
容器的端口映射，`-P` 是随机端口，`-p` 是指定端口：
```shell
# docker run -d -it -P --name nginx1.14 nginx:1.14.2 
# docker port  nginx1.14 
80/tcp -> 0.0.0.0:32768
# docker run -d -it -p 80:80 --name nginx1.14_2 nginx:1.14.2 
# docker port nginx1.14_2 
80/tcp -> 0.0.0.0:80
```
启动容器使用 `--restart` 选项：
```shell
# docker run -it -d -P --restart=always --name nginx1.14_3 nginx:1.14.2 
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
b06d8c9543f2        nginx:1.14.2        "nginx -g 'daemon of…"   14 minutes ago      Up 4 minutes        0.0.0.0:32769->80/tcp   nginx1.14_3
9539d958f31d        nginx:1.14.2        "nginx -g 'daemon of…"   3 hours ago         Up 2 seconds        0.0.0.0:80->80/tcp      nginx1.14_2
# systemctl restart docker
# docker ps               
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
b06d8c9543f2        nginx:1.14.2        "nginx -g 'daemon of…"   14 minutes ago      Up 1 second         0.0.0.0:32768->80/tcp   nginx1.14_3
```
### 启动/关闭/重启容器 `start` `stop` `kill` `restart`
```shell
# docker start nginx1.14_2 
# docker stop nginx1.14_3
# docker restart nginx1.14_2
# docker kill nginx1.14_2
# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
### 暂停/取消暂停 `pause` `unpause`
```shell
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
b06d8c9543f2        nginx:1.14.2        "nginx -g 'daemon of…"   19 minutes ago      Up 5 seconds        0.0.0.0:32769->80/tcp   nginx1.14_3
# docker pause  nginx1.14_3 
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                   PORTS                   NAMES
b06d8c9543f2        nginx:1.14.2        "nginx -g 'daemon of…"   19 minutes ago      Up 19 seconds (Paused)   0.0.0.0:32769->80/tcp   nginx1.14_3
# docker unpause nginx1.14_3 
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
b06d8c9543f2        nginx:1.14.2        "nginx -g 'daemon of…"   21 minutes ago      Up About a minute   0.0.0.0:32769->80/tcp   nginx1.14_3
```
### 容器的创建 `create`
通过 `create` 创建一个新容器，创建的容器处于 `Created` 状态，创建的参数和 `run`，命令的参数相同。通过 `create` 创建的容器需要通过 `start` 命令来启动容器。
### 进入正在运行的容器 `exec`
```shell
# docker exec -it nginx1.14_3  bash
```
### 列出容器 `ps`
可以使用 `docker ps` 进行查看运行容器，也可以使用 `docker container ls ` 进行查看。
```shell
# docker ps     #显示正在运行的容器
# docker ps -a  #显示所有容器
# docker ps -aq -f status=exited #显示状态为 exited 的容器并只显示 CONTAINER ID
# docker ps -l   #显示最近创建的容器
# docker ps -n 3 #显示最近创建的三个容器
```
[详细使用请参考](https://docs.docker.com/engine/reference/commandline/ps/)
### 获取容器元数据 `inspect`
```shell
# docker inspect nginx1.14_3 
# docker inspect -f '{{.NetworkSettings.Networks.bridge.IPAddress}}' nginx1.14_3 # 获取指定字段的信息
```
### 查看容器运行进程 `top`
```shell
# docker top nginx1.14_3 
```
### 获取服务端的实时事件 `events`
[官方文档](https://docs.docker.com/engine/reference/commandline/events/)
### 获取容器日志 `logs`
```shell
# docker logs  nginx1.14_3 # 获取实时日志
```
### 导出容器 `export`
```shell
# docker export -o nginx1.14_3.tar nginx1.14_3
```
### 查看运行容器端口映射 `port`
```shell
# docker port nginx1.14_3 
80/tcp -> 0.0.0.0:32768
```
### 容器与主机文件拷贝 `cp`
```shell
# echo "This is cp file." >index.html
# docker cp index.html nginx1.14_3:/usr/share/nginx/html/
# curl  http://192.168.120.73:32768 
This is cp file.
# curl http://172.17.0.2
This is cp file.
```
### 修改容器名称 `rename`
```shell
# docker rename nginx1.14_3 nginx1.14.2_3
```
### 实时显示容器资源使用 `stats`
```shell
# docker stats
```
### 删除暂停和不使用的资源 `system  prune`
谨慎使用删除命令。
```shell
# docker system  prune 
WARNING! This will remove:
        - all stopped containers
        - all networks not used by at least one container
        - all dangling images
        - all dangling build cache
Are you sure you want to continue? [y/N] 
```