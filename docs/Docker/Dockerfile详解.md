## Dockerfile

### 格式

Dockerfile 一般包含如下几个部分：

* 基础镜像：以哪个镜像作为基础进行制作，用法是FROM 基础镜像名称
* 维护者信息：需要写下该Dockerfile编写人的姓名或邮箱，用法是MANITAINER 名字/邮箱
* 镜像操作命令：对基础镜像要进行的改造命令，比如安装新的软件，进行哪些特殊配置等，常见的是RUN 命令
* 容器启动命令：当基于该镜像的容器启动时需要执行哪些命令，常见的是CMD 命令或ENTRYPOINT

### 使用总结

- 编写.dockerignore 文件
- 容器只运行单个应用
- 将多个 RUN 指令合并为一个
- 基础镜像的标签不要用 latest
- 每个 RUN 指令后删除多余文件
- 选择合适的基础镜像(alpine 版本最好)
- 设置 WORKDIR 和 CMD
- 使用 ENTRYPOINT (可选)
- 在 entrypoint 脚本中使用 exec
- COPY 与 ADD 优先使用前者
- 合理调整 COPY 与 RUN 的顺序
- 设置默认的环境变量，映射端口和数据卷
- 使用 LABEL 设置镜像元数据
- 添加 HEALTHCHECK

### 指令学习

#### FROM

FROM指令是最重要的一个且必须为Dockerfile文件开篇的第一个非注释行，用于为映像文件构建过程指定基准镜像，后续的指令运行基于此基准镜像所提供的运行环境。

实际构建中基准镜像可以是任何可用镜像文件，默认情况下，docker build 会在docker主机上查找指定的镜像文件。如若不存在，则会从Docker Hub Registry 上拉取所需的镜像。如果找不到指定的镜像，docker build 会返回一个错误信息。

> 格式

```dockerfile
FROM <image> [AS <name>]
FROM <image>[:<tag>] [AS <name>]
FROM <image>[@<digest>] [AS <name>]
```

#### MAINTAINIER(deprecated)

用于让Dokcerfile制作者提供本人的详细信息。Dockerfile并不限制MAINTAINER指令出现的位置，但推荐将其放置在FROM指令之后。由于该指令属于遗弃指令，官方建议使用LABEL替代该指令。

> 格式

```dockerfile
MAINTAINER <name>
```

如果使用LABEL描述：

```dockerfile
LABEL maintainer="SvenDowideit@home.org.au"
```

#### LABEL

LABEL指令是向镜像添加元数据。LABEL是一对键值对。要在要在LABEL值中包含空格。可以像在命令行解析中那样使用引号和反斜杠。

> 举例

```dockerfile
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```

#### COPY

COPY指令用于从Docker主机复制文件至创建的新镜像文件中。

> 格式

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

* \<src\>：要复制的源文件或目录，支持使用通配符。
* \<dest\>：目标路径，即正在创建的镜像文件系统路径，通常使用绝对路径。

> 注意事项

* `src`：必须是build上下文中的路径，不能是其父目录中的文件
* 如果 `src` 是目录，则其内部文件或子目录会被递归复制，但`src` 目录自身不会被复制。
* 如果指定了多个`src`，或在`src`中使用了通配符，则`dest` 必须是一个目录，且必须以`/`结尾
* 如果`dest`事先不存在，他将会被自动创建，这包括其父目录路径

> 实践

复制单个文件：

```shell
# ls
Dockerfile  index.html
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
# cat index.html 
Busybox httpd server.
# docker build -t busybox_httpd:1.0 .
Sending build context to Docker daemon  3.072kB
Step 1/3 : FROM busybox:latest
 ---> db8ee88ad75f
Step 2/3 : LABEL maintainer="example <example@example.com>"
 ---> Running in 458d674411a9
Removing intermediate container 458d674411a9
 ---> dc2f17203702
Step 3/3 : COPY index.html /data/web/html/
 ---> 5effe63783cb
Successfully built 5effe63783cb
Successfully tagged busybox_httpd:1.0
# docker images |grep busybox_httpd
busybox_httpd       1.0                 5effe63783cb        54 seconds ago      1.22MB
# docker run --name busybox_1 --rm busybox_httpd:1.0 cat /data/web/html/index.html
Busybox httpd server.
```

复制目录：

```shell
# cp -r /etc/yum.repos.d .
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/
# docker build -t busybox_httpd:1.1 .
# docker run --rm --name buxybox_1 busybox_httpd:1.1 ls /etc/yum.repos.d/
CentOS-Base.repo
bak
docker-ce.repo
epel.repo
# docker images |grep busybox_httpd
busybox_httpd       1.1                 59e1c09bf390        2 minutes ago       1.24MB
busybox_httpd       1.0                 5effe63783cb        13 hours ago        1.22MB
```

#### ADD

ADD指令类是于COPY指令，但是ADD指令支持使用tar文件和URL路径。

> 格式

```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

> 注意事项

* 如果`src`为URL且`dest`不以`/`结尾，则`src`指定的文件将被下载并直接创建为`dest`；如果`dest`以`/`结尾，则文件名URL指定的文件将被直接下载并保存为`dest/filename`。
* 如果`src`是一个本地系统上的压缩格式tar文件，他将被展开为一个目录，其行为类是于`tar -xf` 命令；然而通过URL获取到的tar文件将不会自动展开。
* 如果`src`有多个，或其间接或直接使用了通配，则`dest`必须是一个以`/`结尾的目录路径；如果`dest`不以`/`结尾，则其被视作一个普通文件，`src`的内容将被直接写入到`dest`;

> 使用举例一

```shell
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/
ADD http://nginx.org/download/nginx-1.14.2.tar.gz /usr/local/src/
# docker build -t busybox_httpd:1.2 .
Sending build context to Docker daemon  1.044MB
Step 1/5 : FROM busybox:latest
 ---> db8ee88ad75f
Step 2/5 : LABEL maintainer="example <example@example.com>"
 ---> Using cache
 ---> dc2f17203702
Step 3/5 : COPY index.html /data/web/html/
 ---> Using cache
 ---> 5effe63783cb
Step 4/5 : COPY yum.repos.d /etc/yum.repos.d/
 ---> Using cache
 ---> 59e1c09bf390
Step 5/5 : ADD http://nginx.org/download/nginx-1.14.2.tar.gz /usr/local/src/
Downloading  1.015MB/1.015MB
 ---> 40b8c1eaa8bd
Successfully built 40b8c1eaa8bd
Successfully tagged busybox_httpd:1.2
# docker run --rm --name busybox_1 busybox_httpd:1.2 ls /usr/local/src/
nginx-1.14.2.tar.gz
```

> 使用举例二

```shell
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/
ADD nginx-1.14.2.tar.gz  /usr/local/src/
# docker build -t busybox_httpd:1.3 .
# docker run --rm --name busybox_1 busybox_httpd:1.3 ls -l /usr/local/src/ 
total 0
drwxr-xr-x    8 1001     1001           158 Dec  4  2018 nginx-1.14.2
```

#### WORKDIR

用于为Dockerfile中所有的RUN、CMD、ENTRYPOINT、COPY、ADD指令设定工作目录。

> 格式

```dockerfile
WORKDIR /path/to/workdir
```

> 注意事项

* 在Dockerfile中，WORKDIR指令可出现多次，其路径也可以为相对路径，不过其是相对比此前一个WORKDIR指定的路径。
* WORKDIR也可以调用由ENV指令定义的变量。

> 使用举例

```shell
# cat Dockerfile
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/
WORKDIR /usr/local/src/
ADD nginx-1.14.2.tar.gz ./
```

#### VOLUME

用于在image中创建一个挂载点目录，以挂载Docker host 上的卷或其它容器上的卷。

> 格式

```dockerfile
VOLUME ["/data"]
```

> 注意事项

* 如果挂载点目录路径下此前在文件存在，docker run 命令会在卷挂载完成后将此前的所有文件复制到新挂载的卷中。

> 使用举例

```shell
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/
WORKDIR /usr/local/src/
ADD nginx-1.14.2.tar.gz ./
VOLUME ["/data/"]
# docker build -t busybox_httpd:1.4 .
# docker run -it  --rm --name busybox_1 busybox_httpd:1.4 sh
/usr/local/src # mount |grep data
/dev/sda3 on /data type xfs (rw,relatime,attr2,inode64,noquota)
# docker inspect busybox_1 
...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "62cfc99aa3880c017aae815ec2821589b94b87893ebe8d9327d45cea64d130f4",
                "Source": "/var/lib/docker/volumes/62cfc99aa3880c017aae815ec2821589b94b87893ebe8d9327d45cea64d130f4/_data",
                "Destination": "/data",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
...
```

#### EXPOSE

用于为容器打开指定要监听的端口以实现与外部通信。

> 格式

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

> 注意事项

* EXPOSE指令可一次指定多个端口。

> 使用举例

```shell
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/
WORKDIR /usr/local/src/
ADD nginx-1.14.2.tar.gz ./
VOLUME ["/data/"]
EXPOSE 80/tcp
# docker build -t busybox_httpd:1.5 .
# docker run  --rm --name busybox_1 busybox_httpd:1.5 /bin/httpd -f -h /data/web/html
# docker inspect -f "{{.NetworkSettings.Networks.bridge.IPAddress}}" busybox_1
172.17.0.2
# curl 172.17.0.2
Busybox httpd server.
# docker port busybox_1
```

上面例子中由于容器启动未指定`-P`和`-p`选项，所以无法通过属主机IP进行访问。

```shell
# docker run -P --rm --name busybox_1 busybox_httpd:1.5 /bin/httpd -f -h /data/web/html
# docker port busybox_1 
80/tcp -> 0.0.0.0:32768
```

#### ENV

用于为镜像定义所需的环境变量，并可被Dockerfile文件中位于其后的其它指令所调用。调用格式为`$variable_name`或`${variable_name}`。

> 格式

```dockerfile
ENV <key> <value>
ENV <key>=<value> ...
```

* 第一种格式中，`key`之后的所有内容均会被视作其`value`的组成部分，因此，一次只能设置一个变量；
* 第二种格式可用一次设置多个变量，每个变量为一个`"key=value"`的键值对，如果`value`中包含空格，可以用反斜线`\`进行转义，也可通过对`value`加引号进行标识。反斜线`\`也可以用于续行。
* 如果定义多个变量，建议使用第二种格式。

> 使用举例

```shell
# cat Dockerfile
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
ENV DOC_ROOT=/data/web/html/ \
    WEB_SERVER_PACKAGE="nginx-1.14.2"
COPY index.html ${DOC_ROOT:-/data/web/html/}
COPY yum.repos.d /etc/yum.repos.d/
WORKDIR /usr/local/src/
ADD ${WEB_SERVER_PACKAGE}.tar.gz ./
VOLUME ["/data/"]
EXPOSE 80/tcp
# docker build -t busybox_httpd:1.6 . 
# docker run --rm --name busybox_1 busybox_httpd:1.6 ls -l  /usr/local/src/ /data/web/html/  
/data/web/html/:
total 4
-rw-r--r--    1 root     root            22 Aug 10 14:21 index.html

/usr/local/src/:
total 0
drwxr-xr-x    8 1001     1001           158 Dec  4  2018 nginx-1.14.2
# docker run --rm --name busybox_1 busybox_httpd:1.6 printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=1f3f27531b50
DOC_ROOT=/data/web/html/
WEB_SERVER_PACKAGE=nginx-1.14.2
HOME=/root
```

#### RUN

用于指定docker build过程中运行的程序，可以是任何命令。

> 格式

```dockerfile
RUN <command>
RUN ["executable", "param1", "param2"]
```

* 第一种格式中，`<command>`通常是一个shell命令，且以"/bin/sh -c" 来运行它，这意味着此进程在容器中的PID不为1，不能接收Unix信号，因此，当使用`docker stop <container>`命令停止容器时，此进程接收不到SIGTERM信号。
* 第二种语法格式的参数是一个JSON格式的数组，其中`<executable>`为要运行的命令，后面的`<paramN>`为传递给命令的选项或参数；然而，此种格式指定的命令不会以"/bin/sh -c"来发起，因此常见的shell操作如变量替换以及通配（?*）替换将不会进行；不过，如果要运行的命令依赖于此shell特性的话，可以将其替换为类是下面的格式。

```dockerfile
RUN["/bin/bash", "-c", "<executable>", "param1"]
```



> 使用举例

```shell
# cat Dockerfile 
FROM busybox:latest
#MAINTAINER "example <example@example.com>"
LABEL maintainer="example <example@example.com>"
ENV DOC_ROOT=/data/web/html/ \
    WEB_SERVER_PACKAGE="nginx-1.14.2"
COPY index.html ${DOC_ROOT:-/data/web/html/}
COPY yum.repos.d /etc/yum.repos.d/
WORKDIR /usr/local/src/
ADD http://nginx.org/download/${WEB_SERVER_PACKAGE}.tar.gz ./
VOLUME ["/data/"]
EXPOSE 80/tcp
RUN cd /usr/local/src/ && tar xf ${WEB_SERVER_PACKAGE}.tar.gz
# docker build -t busybox_httpd:1.7 .
# docker run --rm --name busybox_1 busybox_httpd:1.7 ls /usr/local/src/
nginx-1.14.2
nginx-1.14.2.tar.gz
```

#### CMD

类是于RUN命令，CMD指令也可用于运行任何命令或者应用程序，不过，二者的运行时间点不同。

* RUN指令运行于镜像文件构建过程中，而CMD指令运行于基于Dockerfile构建出的新镜像文件启动一个容器时。
* CMD指令的首要目的在于为启动的容器指定默认要运行的程序，且其运行结束后，容器也将终止。不过，CMD指令的命令可以被docker run 的命令选项覆盖。
* 在Dockerfile中可以存在多个CMD指令，但仅最后一个会生效

> 格式

```dockerfile
CMD ["executable","param1","param2"] (exec form, this is the preferred form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
CMD command param1 param2 (shell form)
```

`CMD ["param1","param2"]` 用于为ENTRYPOINT指令提供默认参数。









https://www.jianshu.com/p/e6cc6c27ea74

https://docs.docker.com/engine/reference/builder/

https://yeasy.gitbooks.io/docker_practice/image/build.html

http://guide.daocloud.io/dcs/dockerfile-9153584.html

https://github.com/zhangpeihao/LearningDocker/blob/master/manuscript/04-WriteDockerfile.md

https://blog.fundebug.com/2017/05/15/write-excellent-dockerfile/

https://www.cnblogs.com/cuchadanfan/p/6306973.html



