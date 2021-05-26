## Pod

### Pod简介

Pod是Kubernetes中能够创建和部署的最小单元，是Kubernetes集群中的一个应用实例，总是部署在同一个节点Node上。它们共享IPC、Network和UTC namespace，并且可以声明共享同一个 Volume，是Kubernetes调度的基本单位。Pod的设计理念是支持多个容器在一个Pod中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

Pod 的运行方式有如下两种：

- 单容器Pod，最常见的应用方式。
- 多容器Pod，对于多容器Pod，Kubernetes会保证所有的容器都在同一台物理主机或虚拟主机中运行。多容器Pod是相对高阶的使用方式，除非应用耦合特别严重，一般不推荐使用这种方式。一个Pod内的容器共享IP地址和端口范围，容器之间可以通过 localhost 互相访问。 

### 实现原理

通过前面的介绍我们我已知道如果Pod中启动多容器，那么他们之间是共享IPC、Network和UTC namespace以及可以声明共享一个Volume。但是这种模式下我们通过docker命令参数指定依赖的容器也是能实现的，例如：

```shell
docker run --net=B --volumes-from=B --name=A image-A ...
```

但是这种方式有一个很大的缺陷，那就是容器B必须必容器A先启动，但是这种操作就导致容器之间不是对等的关系，而是拓扑的关系。

所以，在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 JoinNetwork Namespace 的方式，与 Infra 容器关联在一起。这样的组织关系，可以用下面这样一个示意图表达：

![](img/infra.png)



如上图所示，这个 Pod 里有两个用户容器 A 和 B，还有一个 Infra 容器。很容易理解，在Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。

而在 Infra 容器 “Hold 住“ Network Namespace 后，用户容器就可以加入到 Infra 容器的Network Namespace 当中了。所以，如果你查看这些容器在宿主机上的Namespace 文件（这个 Namespace 文件的路径，我已经在前面的内容中介绍过），它们指向的值一定是完全一样的。

这也就意味着，对于 Pod 里的容器 A 和容器 B 来说：

* 它们可以直接使用 localhost 进行通信；
* 它们看到的网络设备跟 Infra 容器看到的完全一样；
* 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
* 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
* Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。

而对于同一个 Pod 里面的所有用户容器来说，它们的进出流量，也可以认为都是通过 Infra 容器完成的。这一点很重要，**因为将来如果你要为 Kubernetes 开发一个网络插件时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。**

有了这个设计之后，共享 Volume 就简单多了：Kubernetes 项目只要把所有 Volume 的定义都设计在 Pod 层级即可。

一个 Volume 对应的宿主机目录对于 Pod 来说就只有一个，Pod 里的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume 对应的宿主机目录。比如下面这个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

在这个例子中，debian-container 和 nginx-container 都声明挂载了 shared-data 这个Volume。而 shared-data 是 hostPath 类型。所以，它对应在宿主机上的目录就是：/data。而这个目录，其实就被同时绑定挂载进了上述两个容器当中。这就是为什么，nginx-container 可以从它的 /usr/share/nginx/html 目录中，读取到 debian-container 生成的 index.html 文件的原因。

### 容器设计模式

Pod 这种“超亲密关系”容器的设计思想，实际上就是希望，当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 Pod 里的多个容器。

#### 使用举例（java web 服务器与 war 包）

我们现在有一个 Java Web 应用的 WAR 包，它需要被放在 Tomcat 的 webapps 目录下运行起来。学习了Pod之后，我们可以把 WAR 包和 Tomcat 分别做成镜像，然后把它们作为一个 Pod 里的两个容器“组合”在一起。这个 Pod 的配置文件如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```

上面的例子中启动了两个容器，第一个容器只是将一个war包放到/app 目录下，第二个容器就是一个tomcat容器。这里一个特殊的地方就是第一个容器是`initContainers`类型的容器。并且两个容器都使用名称为`app-volume`的Volume做数据共享。

> 在 Pod 中，所有 Init Container 定义的容器。都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

像这样，我们就用一种“组合”方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题。这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：sidecar。顾名思义，sidecar 指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。

比如，在我们的这个应用 Pod 中，Tomcat 容器是我们要使用的主容器，而 WAR 包容器的存在，只是为了给它提供一个 WAR 包而已。所以，我们用Init Container 的方式优先运行 WAR 包容器，扮演了一个 sidecar 的角色。

#### 使用举例（容器的日志收集）

我现在有一个应用，需要不断地把日志文件输出到容器的 /var/log 目录中。

1. 可以把一个 Pod 里的 Volume 挂载到应用容器的 /var/log 目录上。
2. 在这个 Pod 里同时运行一个 sidecar 容器，它也声明挂载同一个 Volume 到自己的/var/log 目录上。
3. 接下来 sidecar 容器就只需要做一件事儿，那就是不断地从自己的 /var/log 目录里读取日志文件，转发到 MongoDB 或者 Elasticsearch 中存储起来。这样，一个最基本的日志收集工作就完成了。

### Pod 生命周期

1. Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
2. Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
3. Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
4. Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
5. Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。

### 总结

> Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。

## Pod 对象

你已经非常清楚：Pod，而不是容器，才是 Kubernetes 项目中的最小编排单位。将这个设计落实到 API 对象上，容器（Container）就成了 Pod 属性里的一个普通的字段。那么，一个很自然的问题就是：到底哪些属性属于 Pod 对象，而又有哪些属性属于Container 呢？而如果你能把 Pod 看成传统环境里的“机器”、把容器看作是运行在这个“机器”里的“用户程序”，那么很多关于 Pod 对象的设计就非常容易理解了。

**凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。**

### Pod 字段

#### NodeSelector

是一个供用户将 Pod 与 Node 进行绑定的字段。

```yaml
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```

这样的一个配置，意味着这个 Pod 永远只能运行在携带了“disktype: ssd”标签（Label）的节点上；否则，它将调度失败。

#### NodeName

一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到。

#### HostAliases

定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容。

```yaml
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

在这个 Pod 的 YAML 文件中，我设置了一组 IP 和 hostname 的数据。这样，这个 Pod 启动后，/etc/hosts 文件的内容将如下所示：

```shell
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

在 Kubernetes 项目中，如果要设置 hosts 文件里的内容，一定要通过这种方法。否则，如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。

#### shareProcessNamespace

这个 Pod 里的容器要共享 PID Namespace。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx:1.14.2
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

下面的操作是在1.14的版本中进行的，这是由于1.11版本中默认未开启shareProcessNamespace参数，具体请参考下面的文档。

[Share Process Namespace between Containers in a Pod](<https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/>)

```yaml
# kubectl apply -f nginx.yaml 
pod/nginx created
# # kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   2/2     Running   0          2m37s
# kubectl attach -it nginx -c shell
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    6 root      0:00 nginx: master process nginx -g daemon off;
   11 101       0:00 nginx: worker process
   12 root      0:00 sh
   17 root      0:00 ps aux
```

可以看到，在这个容器里，我们不仅可以看到它本身的 ps aux指令，还可以看到 nginx 容器的进程，以及 Infra 容器的 /pause 进程。这就意味着，整个 Pod 里的每个容器的进程，对于所有容器来说都是可见的：它们共享了同一个 PID Namespace。

####  hostIPC 、hostNetwork、hostPID

类似地，凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义，比如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

在这个 Pod 中，我定义了共享宿主机的 Network、IPC 和 PID Namespace。这就意味着，这个 Pod 里的所有容器，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程。

#### initContainers

该字段属于Pod对容器的定义，只是initContainers的生命周期会先于所有的 Containers 字段，并且严格按照定义的顺序执行。

#### ImagePullPolicy

定义了镜像拉取的策略。而它之所以是一个 Container 级别的属性，是因为容器镜像本来就是Container 定义中的一部分。

* Always：不管镜像是否存在都会进行一次拉取。
* Never：不管镜像是否存在都不会进行拉取
* IfNotPresent：只有镜像不存在时，才会进行镜像拉取。

注意事项：

* 默认为 IfNotPresent ，但 :latest 标签的镜像默认为 Always 。
* 拉取镜像时docker会进行校验，如果镜像中的MD5码没有变，则不会拉取镜像数
  据。
* 生产环境中应该尽量避免使用 :latest 标签，而开发环境中可以借助 :latest 标签自动拉取最新的镜像。

#### RestartPoliy

Kubernetes 里的Pod 恢复机制，也叫 restartPolicy。

* Always：在任何情况下，只要容器不在运行状态，就自动重启容器； 

* OnFailure: 只在容器 异常时才自动重启容器；

* Never: 从来不重启容器。

注意，这里的重启是指在Pod所在Node上面本地重启，并不会调度到其他Node上去。

#### lifecyle

它定义的是 Container Lifecycle Hooks。顾名思义，Container Lifecycle Hooks 的作用，是在容器状态发生变化时触发一系列“钩子”。我们来看这样一个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

proStart：它指的是，在容器启动后，立刻执行一个指定的操作。需要明确的是，postStart 定义的操作，虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。当然，如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。

preStop：它发生的时机，则是容器被杀死之前（比如，收到了SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟postStart 不一样。

钩子的回调函数支持两种方式：

* exec：在容器内执行命令
* httpGet：向指定URL发起GET请求

#### LivenessProbe、ReadinessProbe

为了确保容器在部署后确实处在正常运行状态，Kubernetes提供了两种探针（Probe，支持exec、tcp和httpGet方式）来探测容器的状态：

* LivenessProbe：探测应用是否处于健康状态，如果不健康则删除重建改容器
* ReadinessProbe：探测应用是否启动完成并且处于正常服务状态，如果不正常则更新容器的状态

#### resources

Kubernetes通过cgroups限制容器的CPU和内存等计算资源，包括requests（请求，调度器保证调度到资源充足的Node上）和limits（上限）等：

* spec.containers[].resources.limits.cpu ：CPU上限，可以短暂超过，容器也不会被停止
* spec.containers[].resources.limits.memory ：内存上限，不可以超过；如果超过，容器可能会被停止或调度到其他资源充足的机器上
* spec.containers[].resources.requests.cpu ：CPU请求，可以超过
* spec.containers[].resources.requests.memory ：内存请求，可以超过；但如果超过，容器可能会在Node内存不足时清理

比如nginx容器请求30%的CPU和56MB的内存，但限制最多只用50%的CPU和128MB的内存：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  containers:
    - image: nginx
      name: nginx
      resources:
        requests:
          cpu: "300m"
          memory: "56Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
```

#### 其他

Pod 配置中还有很多字段，可以在 [Github](<https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api/core/v1/types.go>) 上搜索 `type PodSpec struct`，进行大致了解一下。同时也可以通过命令行查看：

```shell
# kubectl explain pod
```

上面是查看pod部分字段，如果继续深入可以这么查看：

```shell
# kubectl explain pod.spec
# kubectl explain pod.metadata
```

## Pod 进阶使用

### Projected Volume（投射数据卷）

在 Kubernetes 中，有几种特殊的 Volume，它们存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。这些特殊 Volume 的作用，是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的。这正是 Projected Volume 的含义。

到目前为止，Kubernetes 支持的 Projected Volume 一共有四种：

1. Secret；
2. ConfigMap； 
3. Downward API；
4. ServiceAccountToken。

#### secret

它的作用，是帮你把 Pod 想要访问的加密数据，存放到 Etcd 中。然后，你就可以通过在 Pod 的容器里挂载 Volume 的方式，访问到这些 Secret 里保存的信息了。

Secret 最典型的使用场景，莫过于存放数据库的Credential 信息，比如下面这个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume 
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

在这个 Pod 中，定义了一个简单的容器。它声明挂载的 Volume，并不是常见的 emptyDir 或者 hostPath 类型，而是 projected 类型。而这个 Volume 的数据来源（sources），则是名为 user 和 pass 的 Secret 对象，分别对应的是数据库的用户名和密码。

这里用到的数据库的用户名、密码，正是以 Secret 对象的方式交给 Kubernetes 保存的。完成这个操作的指令，如下所示：

```shell
# cat username.txt 
admin
# cat password.txt 
passwd0w!
# kubectl create secret generic user --from-file=./username.txt
secret/user created
# kubectl create secret generic pass --from-file=./password.txt
secret/pass created
# kubectl get secrets
NAME                  TYPE                                  DATA      AGE
pass                  Opaque                                1         22s
user                  Opaque                                1         40s
```

下面是通过yaml 文件进行创建：

```shell
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: cGFzc3dkMHch
```

通过yaml文件进行创建需要对认证数据做base64转码。

```shell
# echo -n  "admin" |base64
YWRtaW4=
# echo -n 'passwd0w!' |base64 
cGFzc3dkMHch
```

通过前面所提供pod的yaml 文件创建：

```shell
# kubectl apply -f test-projected-volume.yaml
pod/test-projected-volume created
# kubectl get pods test-projected-volume
NAME                    READY     STATUS    RESTARTS   AGE
test-projected-volume   1/1       Running   0          7m
# kubectl exec -it test-projected-volume -- /bin/sh        
/ # cat /projected-volume/username.txt 
admin
/ # cat /projected-volume/password.txt 
passwd0w!
```

从返回结果中，我们可以看到，保存在 Etcd 里的用户名和密码信息，已经以文件的形式出现在了容器的 Volume 目录里。更重要的是，像这样通过挂载方式进入到容器里的 Secret，一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新。其实，这是 kubelet 组件在定时维护这些 Volume。

需要注意的是，这个更新可能会有一定的延时。所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑。

#### configmap

与 Secret 类似的是 ConfigMap，它与 Secret 的区别在于，ConfigMap 保存的是不需要加密的、应用所需的配置信息。而 ConfigMap 的用法几乎与 Secret 完全相同：你可以使用 kubectl create configmap 从文件或者目录创建 ConfigMap，也可以直接编写 ConfigMap 对象的 YAML 文件。

比如，一个 Java 应用所需的配置文件（.properties 文件），就可以通过下面这样的方式保存在 ConfigMap 里：

```shell
# cat ui.properties 
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
# kubectl create configmap ui-config --from-file=./ui.properties
configmap/ui-config created
# kubectl get configmaps ui-config -o yaml
apiVersion: v1
data:
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: 2019-08-13T10:42:55Z
  name: ui-config
  namespace: default
  resourceVersion: "313245"
  selfLink: /api/v1/namespaces/default/configmaps/ui-config
  uid: 1c14f7d3-bdb7-11e9-8762-000c2999e480
```

#### Downward API

它的作用是：让 Pod 里的容器能够直接获取到这个 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

上面的例子中声明的projected类型的volume，其数据来源为downwardAPI。在downwardAPI中声明暴露Pod的metadata.labels信息给容器。通过这种方式当前Pod的labels字段的值，就会被 Kubernetes 自动挂载成为容器里的/etc/podinfo/labels 文件。而这个容器的启动命令，则是不断打印出 /etc/podinfo/labels 里的内容。

```shell
# kubectl create -f test-downwardapi-volume.yaml
# kubectl logs test-downwardapi-volume


cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
...
```

说明：上面使用的版本是是1.14。

下面列举部分downwardAPI支持的字段：

```shell
1. 使用 fieldRef 可以声明使用:
spec.nodeName - 宿主机名字
status.hostIP - 宿主机 IP
metadata.name - Pod 的名字
metadata.namespace - Pod 的 Namespace
status.podIP - Pod 的 IP
spec.serviceAccountName - Pod 的 Service Account 的名字
metadata.uid - Pod 的 UID
metadata.labels['<KEY>'] - 指定 <KEY> 的 Label 值
metadata.annotations['<KEY>'] - 指定 <KEY> 的 Annotation 值
metadata.labels - Pod 的所有 Label
metadata.annotations - Pod 的所有 Annotation

2. 使用 resourceFieldRef 可以声明使用:
容器的 CPU limit
容器的 CPU request
容器的 memory limit
容器的 memory request
```

需要注意的是，Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果你想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的 PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 sidecar 容器。

Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，通过环境变量获取这些信息的方式，不具备自动更新的能力。所以，一般情况下，我都建议你使用 Volume 文件的方式获取这些信息。

#### ServiceAccountToken

Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是Kubernetes 进行权限分配的对象。像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作ServiceAccountToken。任何运行在 Kubernetes 集群上的应用，都必须使用这个ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server。

如果你查看一下任意一个运行在 Kubernetes 集群里的 Pod，就会发现，每一个 Pod，都已经自动声明一个类型是 Secret、名为 default-token-xxxx 的 Volume，然后 自动挂载在每个容器的一个固定目录上。比如：

```shell
# kubectl describe pod nginx-deployment-69bd8b4999-g87wn 
...
    Mounts:
      /usr/share/nginx/html from nginx-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-2n8nl (ro)
...
Volumes:
  nginx-vol:
    Type:          HostPath (bare host directory volume)
    Path:          /var/data
    HostPathType:  
  default-token-2n8nl:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-2n8nl
    Optional:    false
```

这个 Secret 类型的 Volume，正是默认 Service Account 对应的 ServiceAccountToken。所以说，Kubernetes 其实在每个 Pod 创建的时候，自动在它的 spec.volumes 部分添加上了默认 ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 volumeMounts 字段。这个过程对于用户来说是完全透明的。

这样，一旦 Pod 创建完成，容器里的应用就可以直接从这个默认 ServiceAccountToken 的挂载目录里访问到授权信息和文件。这个容器内的路径在 Kubernetes 里是固定的，即：/var/run/secrets/kubernetes.io/serviceaccount ，而这个 Secret 类型的 Volume 里面的内容如下所示：

```shell
# kubectl exec -it nginx-deployment-69bd8b4999-g87wn -- /bin/bash
viceaccountdeployment-69bd8b4999-g87wn:/# ls  /var/run/secrets/kubernetes.io/ser 
ca.crt  namespace  token
```

所以，你的应用程序只要直接加载这些授权文件，就可以访问并操作 Kubernetes API 了。而且，如果你使用的是 Kubernetes 官方的 Client 包（k8s.io/client-go）的话，它还可以自动加载这个目录下的文件，你不需要做任何配置或者编码操作。

这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”。

### 容器的健康检查和恢复机制

在 Kubernetes 中，你可以为 Pod 里的容器定义一个健康检查“探针”（Probe）。这样，kubelet 就会根据这个 Probe 的返回值决定这个容器的状态，而不是直接以容器进行是否运行（来自 Docker 返回的信息）作为依据。这种机制，是生产环境中保证应用健康存活的重要手段。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

在这个Pod中，它在启动之后做的第一件事，就是在 /tmp 目录下创建了一个 healthy 文件，以此作为自己已经正常运行的标志。而 30 s 过后，它会把这个文件删除掉。

与此同时，我们定义了一个这样的 livenessProbe（健康检查）。它的类型是 exec，这意味着，它会在容器启动后，在容器里面执行一句我们指定的命令，比如：“cat /tmp/healthy”。这时，如果这个文件存在，这条命令的返回值是 0，Pod 就会认为这个容器不仅已经启动，而且是健康的。这个健康检查，在容器启动 5 s 后开始执行（initialDelaySeconds: 5），每 5 s 执行一次（periodSeconds: 5）。

下面使创建这个Pod并查看Pod的状态：

```shell
# kubectl create -f test-liveness-exec.yaml
pod/test-liveness-exec created
# kubectl get pods test-liveness-exec
NAME                 READY     STATUS    RESTARTS   AGE
test-liveness-exec   1/1       Running   0          8s
# kubectl describe pod test-liveness-exec
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  29s   default-scheduler  Successfully assigned default/test-liveness-exec to c7-node1
  Normal  Pulling    29s   kubelet, c7-node1  Pulling image "busybox"
  Normal  Pulled     28s   kubelet, c7-node1  Successfully pulled image "busybox"
  Normal  Created    28s   kubelet, c7-node1  Created container liveness
  Normal  Started    27s   kubelet, c7-node1  Started container liveness
```

过一段时间之后在查看Pod的状态：

```shell
# kubectl get pod test-liveness-exec
NAME                 READY   STATUS    RESTARTS   AGE
test-liveness-exec   1/1     Running   3          3m46s
# kubectl describe pod test-liveness-exec
...
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  4m48s                  default-scheduler  Successfully assigned default/test-liveness-exec to c7-node1
  Normal   Pulled     2m19s (x3 over 4m47s)  kubelet, c7-node1  Successfully pulled image "busybox"
  Normal   Created    2m19s (x3 over 4m47s)  kubelet, c7-node1  Created container liveness
  Normal   Started    2m19s (x3 over 4m46s)  kubelet, c7-node1  Started container liveness
  Warning  Unhealthy  95s (x9 over 4m15s)    kubelet, c7-node1  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    95s (x3 over 4m5s)     kubelet, c7-node1  Container liveness failed liveness probe, will be restarted
  Normal   Pulling    65s (x4 over 4m48s)    kubelet, c7-node1  Pulling image "busybox"
```

这时我们发现，Pod 并没有进入 Failed 状态，而是保持了 Running 状态。这是由于这个异常的容器已经被 Kubernetes 重启了。在这个过程中，Pod 保持 Running 状态不变。

**Kubernetes 中并没有 Docker 的 Stop 语义。所以虽然是 Restart（重启），但实际却是重新创建了容器。**

这个功能就是 Kubernetes 里的Pod 恢复机制，也叫 restartPolicy。它是 Pod 的 Spec 部分的一个标准字段（pod.spec.restartPolicy），默认值是 Always，即：任何时候这个容器发生了异常，它一定会被重新创建。

Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。而如果你想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理 Pod，你只需要一个 Pod 副本。

关于 restartPolicy 使用的注意事项：

1. 只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个Pod 就会保持 Running 状态，并进行容器重启。否则，Pod 就会进入 Failed 状态 。
2. 对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数，比如：

```shell
# kubectl get pods test-liveness-exec
NAME                 READY   STATUS    RESTARTS   AGE
test-liveness-exec   1/1     Running   0          8s
```

除了在容器中执行命令外，livenessProbe 也可以定义为发起 HTTP 或者 TCP 请求的方式，定义格式如下：

```yaml
...
livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: X-Custom-Header
         value: Awesome
       initialDelaySeconds: 3
       periodSeconds: 3
```

```yaml
    ...
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

在 Kubernetes 的 Pod 中，还有一个叫 readinessProbe 的字段。虽然它的用法与 livenessProbe 类似，但作用却大不一样。readinessProbe 检查结果的成功与否，决定的这个Pod 是不是能被通过 Service 的方式访问到，而并不影响 Pod 的生命周期。这部分内容在 Service 中举例。

### Pod 自动填充字段（PodPreset）

开启kube-apiserver 的PodPreset功能：

```shell
# cat /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
  ...
    - --runtime-config=api/all=true,settings.k8s.io/v1alpha1=true
    - --enable-admission-plugins=PodPreset
...
# kubectl restart kubelet
```

提供一个基础功能pod的yaml 文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

定义一个PodPreset对象文件：

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

创建Pod：

```shell
# kubectl create -f preset.yaml 
podpreset.settings.k8s.io/allow-database created
# kubectl create -f website.yaml 
pod/website created
```

查看Pod 的API对象：

```shell
# kubectl get pod website -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    podpreset.admission.kubernetes.io/podpreset-allow-database: "48973"
  creationTimestamp: "2019-08-15T08:18:08Z"
  labels:
    app: website
    role: frontend
  name: website
  namespace: default
  resourceVersion: "49031"
  selfLink: /api/v1/namespaces/default/pods/website
  uid: 36c8eda2-bf35-11e9-9f14-000c29b3910e
spec:
  containers:
  - env:
    - name: DB_PORT
      value: "6379"
    image: nginx:1.14.2
    imagePullPolicy: IfNotPresent
    name: website
    ports:
    - containerPort: 80
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-46wf4
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: c7-node1
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - emptyDir: {}
    name: cache-volume
  - name: default-token-46wf4
    secret:
      defaultMode: 420
      secretName: default-token-46wf4
status:
  conditions:
...
```

我们也可以在命令行查看PodPreset：

```shell
# kubectl get podpreset
NAME             CREATED AT
allow-database   2019-08-15T08:17:40Z
# kubectl get podpreset -o yaml
apiVersion: v1
items:
- apiVersion: settings.k8s.io/v1alpha1
  kind: PodPreset
  metadata:
    creationTimestamp: "2019-08-15T08:17:40Z"
    generation: 1
    name: allow-database
    namespace: default
    resourceVersion: "48973"
    selfLink: /apis/settings.k8s.io/v1alpha1/namespaces/default/podpresets/allow-database
    uid: 264f53f1-bf35-11e9-9f14-000c29b3910e
  spec:
    env:
    - name: DB_PORT
      value: "6379"
    selector:
      matchLabels:
        role: frontend
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
    volumes:
    - emptyDir: {}
      name: cache-volume
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

通过上面的离职，我们可以看到这个 Pod 里多了新添加的 labels、env、volumes 和 volumeMount 的定义，它们的配置跟 PodPreset 的内容一样。此外，这个 Pod 还被自动加上了一个 annotation 表示这个 Pod 对象被 PodPreset 改动过。

**请注意：PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。**

比如，我们现在提交的是一个 nginx-deployment，那么这个 Deployment 对象本身是永远不会被 PodPreset 改变的，被修改的只是这个 Deployment 创建出来的所有 Pod。这一点请务必区分清楚。

如果你定义了同时作用于一个 Pod 对象的多个PodPreset，Kubernetes 项目会帮你合并（Merge）这两个 PodPreset 要做的修改。而如果它们要做的修改有冲突的话，这些冲突字段就不会被修改。