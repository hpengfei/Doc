## Kubernetes

### 简介

Kubernetes是谷歌开源的容器集群管理系统，是Google多年大规模容器管理技术Borg的开源版本，主要功能包括：

- 基于容器的应用部署、维护和滚动升级
- 负载均衡和服务发现
- 跨机器和跨地区的集群调度
- 自动伸缩
- 无状态服务和有状态服务
- 广泛的Volume支持
- 插件机制保证扩展性

## 架构

![](img/framework.png)

核心组件介绍：

- etcd：保存了整个集群的状态；
- apiserver：提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler：负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet：负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理；
- Container runtime：负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy：负责为Service提供cluster内部的服务发现和负载均衡；

常用插件介绍：

- CoreDNS：负责为整个集群提供DNS服务
- Ingress Controller：为服务提供外网入口
- Prometheus：提供资源监控
- Dashboard：提供GUI
- Federation：提供跨可用区的集群

## Kubernetes 的基本概念

### Pod

Pod是在Kubernetes集群中运行部署应用或服务的最小单元，它是可以支持多容器的。Pod的设计理念是支持多个容器在一个Pod中共享网络地址和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。Pod对多容器的支持是K8最基础的设计理念。

目前Kubernetes中的业务主要可以分为长期伺服型（long-running）、批处理型（batch）、节点后台支撑型（node-daemon）和有状态应用型（stateful application）；分别对应的小机器人控制器为Deployment、Job、DaemonSet和StatefulSet。

### 副本控制器（Replication Controller，RC）

RC是Kubernetes集群中最早的保证Pod高可用的API对象。通过监控运行中的Pod来保证集群中运行指定数目的Pod副本。指定的数目可以是多个也可以是1个；少于指定数目，RC就会启动运行新的Pod副本；多于指定数目，RC就会杀死多余的Pod副本。即使在指定数目为1的情况下，通过RC运行Pod也比直接运行Pod更明智，因为RC也可以发挥它高可用的能力，保证永远有1个Pod在运行。RC是Kubernetes较早期的技术概念，只适用于长期伺服型的业务类型，比如控制小机器人提供高可用的Web服务。

### 副本集（Replica Set，RS）

RS是新一代RC，提供同样的高可用能力，区别主要在于RS后来居上，能支持更多种类的匹配模式。副本集对象一般不单独使用，而是作为Deployment的理想状态参数使用。

### 部署 Deployment

部署表示用户对Kubernetes集群的一次更新操作。部署是一个比RS应用模式更广的API对象，可以是创建一个新的服务，更新一个新的服务，也可以是滚动升级一个服务。滚动升级一个服务，实际是创建一个新的RS，然后逐渐将新RS中副本数增加到理想状态，将旧RS中的副本数减小到0的复合操作；这样一个复合操作用一个RS是不太好描述的，所以用一个更通用的Deployment来描述。以Kubernetes的发展方向，未来对所有长期伺服型的的业务的管理，都会通过Deployment来管理。

### Service

RC、RS和Deployment只是保证了支撑服务的微服务Pod的数量，但是没有解决如何访问这些服务的问题。一个Pod只是一个运行服务的实例，随时可能在一个节点上停止，在另一个节点以一个新的IP启动一个新的Pod，因此不能以确定的IP和端口号提供服务。要稳定地提供服务需要服务发现和负载均衡能力。服务发现完成的工作，是针对客户端访问的服务，找到对应的的后端服务实例。在K8集群中，客户端需要访问的服务就是Service对象。每个Service会对应一个集群内部有效的虚拟IP，集群内部通过虚拟IP访问一个服务。在Kubernetes集群中微服务的负载均衡是由Kube-proxy实现的。Kube-proxy是Kubernetes集群内部的负载均衡器。它是一个分布式代理服务器，在Kubernetes的每个节点上都有一个；这一设计体现了它的伸缩性优势，需要访问服务的节点越多，提供负载均衡能力的Kube-proxy就越多，高可用节点也随之增多。与之相比，我们平时在服务器端做个反向代理做负载均衡，还要进一步解决反向代理的负载均衡和高可用问题。

### 任务 （Job）

Job是Kubernetes用来控制批处理型任务的API对象。批处理业务与长期伺服业务的主要区别是批处理业务的运行有头有尾，而长期伺服业务在用户不停止的情况下永远运行。Job管理的Pod根据用户的设置把任务成功完成就自动退出了。成功完成的标志根据不同的spec.completions策略而不同：单Pod型任务有一个Pod成功就标志完成；定数成功型任务保证有N个任务全部成功；工作队列型任务根据应用确认的全局成功而标志成功。

### 后台支撑服务集（DaemonSet）

长期伺服型和批处理型服务的核心在业务应用，可能有些节点运行多个同类业务的Pod，有些节点上又没有这类Pod运行；而后台支撑型服务的核心关注点在Kubernetes集群中的节点（物理机或虚拟机），要保证每个节点上都有一个此类Pod运行。节点可能是所有集群节点也可能是通过nodeSelector选定的一些特定节点。典型的后台支撑型服务包括，存储，日志和监控等在每个节点上支持Kubernetes集群运行的服务。

### 有状态服务集（StatefulSet）

RC和RS主要是控制提供无状态服务的，其所控制的Pod的名字是随机设置的，一个Pod出故障了就被丢弃掉，在另一个地方重启一个新的Pod，名字变了。名字和启动在哪儿都不重要，重要的只是Pod总数；而StatefulSet是用来控制有状态服务，StatefulSet中的每个Pod的名字都是事先确定的，不能更改。

适合于StatefulSet的业务包括数据库服务MySQL和PostgreSQL，集群化管理服务ZooKeeper、etcd等有状态服务。StatefulSet的另一种典型应用场景是作为一种比普通容器更稳定可靠的模拟虚拟机的机制。传统的虚拟机正是一种有状态的宠物，运维人员需要不断地维护它，容器刚开始流行时，我们用容器来模拟虚拟机使用，所有状态都保存在容器里，而这已被证明是非常不安全、不可靠的。使用StatefulSet，Pod仍然可以通过漂移到不同节点提供高可用，而存储也可以通过外挂的存储来提供高可靠性，StatefulSet做的只是将确定的Pod与确定的存储关联起来保证状态的连续性。

### 存储卷（Volume）

Kubernetes集群中的存储卷跟Docker的存储卷有些类似，只不过Docker的存储卷作用范围为一个容器，而Kubernetes的存储卷的生命周期和作用范围是一个Pod。每个Pod中声明的存储卷由Pod中的所有容器共享。Kubernetes支持非常多的存储卷类型，特别的，支持多种公有云平台的存储，包括AWS，Google和Azure云；支持多种分布式存储包括GlusterFS和Ceph；也支持较容易使用的主机本地目录emptyDir, hostPath和NFS。Kubernetes还支持使用Persistent Volume Claim即PVC这种逻辑存储，使用这种存储，使得存储的使用者可以忽略后台的实际存储技术（例如AWS，Google或GlusterFS和Ceph），而将有关存储实际技术的配置交给存储管理员通过Persistent Volume来配置。

### 持久存储卷（Persistent Volume，PV）和持久存储卷声明（Persistent Volume Claim，PVC）

PV和PVC使得Kubernetes集群具备了存储的逻辑抽象能力，使得在配置Pod的逻辑里可以忽略对实际后台存储技术的配置，而把这项配置的工作交给PV的配置者，即集群的管理者。存储的PV和PVC的这种关系，跟计算的Node和Pod的关系是非常类似的；PV和Node是资源的提供者，根据集群的基础设施变化而变化，由Kubernetes集群管理员配置；而PVC和Pod是资源的使用者，根据业务服务的需求变化而变化，有Kubernetes集群的使用者即服务的管理员来配置。

### 节点（Node）

Kubernetes集群中的计算能力由Node提供，最初Node称为服务节点Minion，后来改名为Node。Kubernetes集群中的Node也就等同于Mesos集群中的Slave节点，是所有Pod运行所在的工作主机，可以是物理机也可以是虚拟机。不论是物理机还是虚拟机，工作主机的统一特征是上面要运行kubelet管理节点上运行的容器。

### 密钥对象（Secret）

Secret是用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。使用Secret的好处是可以避免把敏感信息明文写在配置文件里。在Kubernetes集群中配置和使用服务不可避免的要用到各种敏感信息实现登录、认证等功能，例如访问AWS存储的用户名密码。为了避免将类似的敏感信息明文写在所有需要使用的配置文件中，可以将这些信息存入一个Secret对象，而在配置文件中通过Secret对象引用这些敏感信息。这种方式的好处包括：意图明确，避免重复，减少暴漏机会。

### 用户帐户（User Account）和服务帐户（Service Account）

用户帐户为人提供账户标识，而服务账户为计算机进程和Kubernetes集群中运行的Pod提供账户标识。用户帐户和服务帐户的一个区别是作用范围；用户帐户对应的是人的身份，人的身份与服务的namespace无关，所以用户账户是跨namespace的；而服务帐户对应的是一个运行中程序的身份，与特定namespace是相关的。

### 命名空间（Namespace）

命名空间为Kubernetes集群提供虚拟的隔离作用，Kubernetes集群初始有两个命名空间，分别是默认命名空间default和系统命名空间kube-system，除此以外，管理员可以可以创建新的命名空间满足需要。

## 资源对象

| 类别     | 名称                                                         |
| -------- | ------------------------------------------------------------ |
| 资源对象 | Pod、ReplicaSet、ReplicationController、Deployment、StatefulSet、DaemonSet、Job、CronJob、HorizontalPodAutoscaling、Node、Namespace、Service、Ingress、Label、CustomResourceDefinition |
| 存储对象 | Volume、PersistentVolume、Secret、ConfigMap                  |
| 策略对象 | SecurityContext、ResourceQuota、LimitRange                   |
| 身份对象 | ServiceAccount、Role、ClusterRole                            |

## Kubernetes API 

每个API对象都有3大类属性：元数据metadata、规范spec和状态status。元数据是用来标识API对象的，每个对象都至少有3个元数据：namespace，name和uid；除此以外还有各种各样的标签labels用来标识和匹配不同的对象，例如用户可以用标签env来标识区分不同的服务部署环境，分别用env=dev、env=testing、env=production来标识开发、测试、生产的不同服务。规范描述了用户期望K8s集群中的分布式系统达到的理想状态（Desired State），例如用户可以通过复制控制器Replication Controller设置期望的Pod副本数为3；status描述了系统实际当前达到的状态（Status），例如系统当前实际的Pod副本数为2；那么复制控制器当前的程序逻辑就是自动启动新的Pod，争取达到副本数为3。

K8s中所有的配置都是通过API对象的spec去设置的，也就是用户通过配置系统的理想状态来改变系统，这是k8s重要设计理念之一，即所有的操作都是声明式（Declarative）的而不是命令式（Imperative）的。声明式操作在分布式系统中的好处是稳定，不怕丢操作或运行多次，例如设置副本数为3的操作运行多次也还是一个结果，而给副本数加1的操作就不是声明式的，运行多次结果就错了。









## Kubernetes 入门操作

命令行操作知识为了初步学习了解Kubernetes的使用，正是使用还是通过yaml文件进行操作Kubernetes。

### 创建一个容器

```shell
# kubectl run --image=nginx:1.14.2 nginx-app --port=80
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx-app created
```

这种创建实际上是一个由deployment来管理的Pod。

### 查看pod

```shell
# kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
nginx-app-7f989cc7f7-kw4t2   1/1     Running   0          95s
# kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   1/1     1            1           99s
# kubectl describe pods nginx-app-7f989cc7f7-kw4t2 
Name:               nginx-app-7f989cc7f7-kw4t2
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               node3/192.168.120.73
Start Time:         Fri, 02 Aug 2019 16:13:27 +0800
Labels:             pod-template-hash=7f989cc7f7
                    run=nginx-app
Annotations:        <none>
Status:             Running
IP:                 10.233.92.27
Controlled By:      ReplicaSet/nginx-app-7f989cc7f7
Containers:
  nginx-app:
    Container ID:   docker://cf80f017aec76407095337247919fb5f12c35871562a20b6a7bbbeb58e34f142
    Image:          nginx:1.14.2
    Image ID:       docker-pullable://nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 02 Aug 2019 16:13:29 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-xwt7d (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-xwt7d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-xwt7d
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m32s  default-scheduler  Successfully assigned default/nginx-app-7f989cc7f7-kw4t2 to node3
  Normal  Pulled     2m31s  kubelet, node3     Container image "nginx:1.14.2" already present on machine
  Normal  Created    2m31s  kubelet, node3     Created container nginx-app
  Normal  Started    2m30s  kubelet, node3     Started container nginx-app
```

### 验证nginx服务

```shell
[root@node2 ~]# curl  -I http://10.233.92.27
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 02 Aug 2019 08:18:34 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes
[root@node1 ~]# kubectl logs -f nginx-app-7f989cc7f7-kw4t2
10.233.96.0 - - [02/Aug/2019:08:18:34 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-
```

### 启动service

在kubernetes中，Pod的IP地址会随着Pod的重启而变化，并不建议直接拿Pod的IP来交互。那如何来访问这些Pod提供的服务呢？使用Service。Service为一组Pod（通过labels来选择）提供一个统一的入口，并为它们提供负载均衡和自动服务发现。可以为前面的 nginx-app 创建一个service：

```shell
# kubectl expose deployment nginx-app --type=NodePort --port=80 --target-port=80 
service/nginx-app exposed
# kubectl describe service nginx-app
Name:                     nginx-app
Namespace:                default
Labels:                   run=nginx-app
Annotations:              <none>
Selector:                 run=nginx-app
Type:                     NodePort
IP:                       10.233.7.86
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30243/TCP
Endpoints:                10.233.92.27:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.233.0.1    <none>        443/TCP        24h
nginx-app    NodePort    10.233.7.86   <none>        80:30243/TCP   10s
```

访问的方式既可以通过pod 的IP 进行访问`http://10.233.92.27`（不建议使用），也可以通过service的 IP地址`http://10.233.7.86`访问，也可以通过node的 IP`http://192.168.120.71:30243`进行访问nginx的资源。

### 扩展应用

```shell
# kubectl scale --replicas=3 deployment nginx-app 
deployment.extensions/nginx-app scaled
# kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   3/3     3            3           41m
# kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
nginx-app-7f989cc7f7-cgns7   1/1     Running   0          25m
nginx-app-7f989cc7f7-kw4t2   1/1     Running   0          66m
nginx-app-7f989cc7f7-rqp4c   1/1     Running   0          25m
```

### 滚动升级

```shell
# kubectl set image deployment/nginx-app nginx-app=nginx:1.17.2
deployment.extensions/nginx-app image updated
# kubectl rollout status deployment nginx-app
Waiting for deployment "nginx-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-app" successfully rolled out
# curl -I http://10.233.7.86
HTTP/1.1 200 OK
Server: nginx/1.17.2
Date: Fri, 02 Aug 2019 09:31:17 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 23 Jul 2019 11:45:37 GMT
Connection: keep-alive
ETag: "5d36f361-264"
Accept-Ranges: bytes
```

### 回滚操作

```shell
# kubectl rollout history deployment nginx-app
deployment.extensions/nginx-app 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
# kubectl rollout undo deployment nginx-app 
deployment.extensions/nginx-app rolled back
# curl -I http://10.233.7.86                 
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 02 Aug 2019 09:43:07 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
Connection: keep-alive
ETag: "5c0692e1-264"
Accept-Ranges: bytes
```

### 资源限制

Kubernetes通过cgroups提供容器资源管理的功能，可以限制每个容器的CPU和内存使用，比如对于刚才创建的deployment，可以通过下面的命令限制nginx容器最多只用50%的CPU和128MB的内存：

```shell
# kubectl set resources deployment nginx-app -c=nginx-app --limits=cpu=500m,memory=128Mi
deployment.extensions/nginx-app resource requirements updated
# kubectl describe pods nginx-app-dbcd7d745-7vz9k
...
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        500m
      memory:     128Mi
...
```





