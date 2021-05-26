## 使用容器需要避免的一些做法
* 不要在容器中保存数据（Don’t store data in containers）
* 将应用打包到镜像再部署而不是更新到已有容器（Don’t ship your application in two pieces）
* 不要产生过大的镜像 （Don’t create large images）
* 不要使用单层镜像 （Don’t use a single layer image）
* 不要从运行着的容器上产生镜像 （Don’t create images from running containers ）
* 不要只是使用 “latest”标签 （Don’t use only the “latest” tag）
* 不要在容器内运行超过一个的进程 （Don’t run more than one process in a single container ）
* 不要在容器内保存 credentials，而是要从外面通过环境变量传入 （ Don’t store credentials in the image. Use environment variables）
* 不要使用 root 用户跑容器进程（Don’t run processes as a root user ）
* 不要依赖于IP地址，而是要从外面通过环境变量传入 （Don’t rely on IP addresses ）
[原文传送](http://developers.redhat.com/blog/2016/02/24/10-things-to-avoid-in-docker-containers/)

## Namespace 资源隔离



| 类型                                           | 系统调用参数  | 内核版本 |
| ---------------------------------------------- | ------------- | -------- |
| Mount Namespace 文件系统 挂载点                | CLONE_NEWNS   | 2.4.19   |
| UTS Namespace 主机名和主机域                   | CLONE_NEWUTS  | 2.6.19   |
| IPC Namespace 信号量、消息队列、共享内存       | CLONE_NEWIPC  | 2.6.19   |
| PID Namespace 进程编号                         | CLONE_NEWPID  | 2.6.24   |
| Network Namespace 网络设备、网络协议栈、端口等 | CLONE_NEWNET  | 2.6.29   |
| User Namespace 操作进程的用户和用户组          | CLONE_NEWUSER | 3.8      |



