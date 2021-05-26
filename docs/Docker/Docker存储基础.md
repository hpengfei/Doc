## Docker 存储资源
### 使用 storage driver 存储
默认情况下，创建容器不指定volume时，容器的数据被保存在容器之内，它只在容器的生命周期内存在，会随着容器的被删除而被删除。可以通过 `docker info` 查看现有环境所使用的 `Storage Driver`。
```shell
# docker info 
......
Server Version: 18.09.6
Storage Driver: overlay2
 Backing Filesystem: xfs
......
```
### 使用 Data volume 存储
Data Volume 本质上是 Docker Host 文件系统中的目录或文件，能够直接被 mount 到容器的文件系统中。Data Volume 有以下特点：

* Data Volume 是目录或文件，而非没有格式化的磁盘（块设备）。
* 容器可以读写 volume 中的数据。
* volume 数据可以被永久的保存，即使使用它的容器已经销毁。

#### 挂载主机目录做为 Data volume
```shell
# cat html/index.html 
This is a file in Docker Host.
# docker run -d -p 80:80 -v /root/html:/usr/share/nginx/html --name website_1 nginx:1.14.2
# docker run -d -p 8080:80 --mount type=bind,source=/root/html,target=/usr/share/nginx/html --name website_2 nginx:1.14.2
# curl localhost:80
This is a file in Docker Host.
# curl localhost:8080
This is a file in Docker Host.
```
挂载目录可以使用 -v 或者 --mount 选项，两者的区别是 --mount 参数时如果本地目录不存在，Docker 会报错。
```shell
# docker run --rm -d -p 8000:80 --mount type=bind,source=/root/aaaa,target=/usr/share/nginx/html --name website_3 nginx:1.14.2   
docker: Error response from daemon: invalid mount config for type "bind": bind source path does not exist: /root/aaaa.
```
Docker 挂载主机目录的默认权限是 读写，用户也可以通过增加 readonly 指定为 只读：
```shell
# docker run -it --rm -d -P --name website_3 -v /root/html:/usr/share/nginx/html:ro nginx:1.14.2
# docker exec -it website_3 bash
root@88f9dfeb4159:~# echo "test" >/usr/share/nginx/html/index.html 
bash: /usr/share/nginx/html/index.html: Read-only file system
```
也可以使用 --mount 参数：
```shell
# docker run -it --rm -d -P --name website_3 --mount type=bind,source=/root/html,target=/usr/share/nginx/html,readonly nginx:1.14.2
```
#### 挂载主机文件做为 Data volume
```shell
# echo "single file test." >html/test.html
# docker run -d -p 9000:80 -it --name website_4 -v /root/html/test.html:/usr/share/nginx/html/website_4_test.html nginx:1.14.2
# curl localhost:9000/website_4_test.html
single file test.
```
#### 创建 Data volume 挂载使用
```shell
# docker volume create my_vol
# docker volume ls
DRIVER              VOLUME NAME
local               my_vol
# docker volume inspect  my_vol 
[
    {
        "CreatedAt": "2019-06-14T14:31:52+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my_vol/_data",
        "Name": "my_vol",
        "Options": {},
        "Scope": "local"
    }
]
# docker run -d -it -p 80 --name website_1 -v my_vol:/usr/share/nginx/html nginx:1.14.2
# docker run -d -it -p 80 --name website_2 --mount source=my_vol,target=/usr/share/nginx/html nginx:1.14.2 
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
3ee34f8a21b9        nginx:1.14.2        "nginx -g 'daemon of…"   5 seconds ago       Up 4 seconds        0.0.0.0:32773->80/tcp   website_2
d581a19f75c6        nginx:1.14.2        "nginx -g 'daemon of…"   5 minutes ago       Up 5 minutes        0.0.0.0:32772->80/tcp   website_1
# echo "my_vol" >/var/lib/docker/volumes/my_vol/_data/index.html
# curl localhost:32772
my_vol
# curl localhost:32773
my_vol
```
通过使用 `docker volume inspect  my_vol` 可以查看到 volume 数据存放位置。容器通过 volume 的持久化保存，不会因为容器的删除而导致volume 中数据的丢失。

注意：删除容器时不建议使用 -v 选择进行删除 volume，避免出现误删除的事故。
```shell
# docker stop website_1
# docker stop website_2
# docker rm website_1
# docker rm website_2
# docker run -d -it -p 80 --name website_3 -v my_vol:/usr/share/nginx/html nginx:1.14.2 
# docker port website_3 
80/tcp -> 0.0.0.0:32774
# curl localhost:32774
my_vol
```
#### 删除 Data volume
Data volume 的删除尽量通过 `docker volume rm` 进行删除，不要在删除容器时进行删除，这边避免出现误删除数据的问题，同时在删除 Data volume 之前需要确认是否还有容器正在使用。
```shell
# docker stop website_3
# docker rm website_3 
# docker volume rm my_vol 
```
如果清理无主的 Data volume 可以通过以下方式：
```shell
# docker volume prune
```
### Data volume 容器共享数据
* 通过容器做共享数据时，共享的容器可以无需启动。
* 命令中没有指定数据卷的信息，也就是说新容器中挂载数据卷的目录和源容器中是一样的。
* 删除挂载了数据卷的容器时，数据卷并不会被自动删除。
* 如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器，然后使用 `docker volume rm` 命令进行删除。
```shell
# docker run -it -v /data --name mydate ubuntu
# docker inspect -f '{{.Mounts}}' mydate       
[{volume a575974acf442ba27f9f7d40ba97d5425245a2d1cd8299d04c8dbbca3798bc57 /var/lib/docker/volumes/a575974acf442ba27f9f7d40ba97d5425245a2d1cd8299d04c8dbbca3798bc57/_data /data local  true }]
# docker run -it -d --volumes-from mydate --name web_1 ubuntu
# docker run -it -d --volumes-from mydate --name web_2 ubuntu
# docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
174d10b3392f        ubuntu              "/bin/bash"         5 minutes ago       Up 5 minutes                            web_2
6f5563f06b1c        ubuntu              "/bin/bash"         5 minutes ago       Up 5 minutes                            web_1
# 在 web_1 容器操作
# docker exec -it web_1 bash
# echo "test" >/data/test.txt
# 在 web_2 容器操作
# docker exec -it web_2 bash
# cat /data/test.txt 
test
```
