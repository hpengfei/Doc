## DaemonSet

### 简介

DaemonSet 的主要作用，是让你在 Kubernetes 集群里，运行一个 Daemon Pod。所以，这个 Pod 有如下三个特征：

1. 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上； 
2. 每个节点上只有一个这样的 Pod 实例；
3. 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

### 常用场景

1. 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；

2. 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；

3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。

### API 使用介绍



```yaml
# cat fluentd-elasticsearch.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: gcr.azk8s.cn/google-containers/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

这个 DaemonSet，管理的是一个 fluentd-elasticsearch 镜像的 Pod。其中镜像的功能为：通过 fluentd 将 Docker 容器里的日志转发到 ElasticSearch 中。DaemonSet 跟 Deployment 其实非常相似，只不过是没有 replicas 字段；它也使用 selector 选择管理所有携带了 name=fluentd-elasticsearch 标签的 Pod。

而这些 Pod 的模板，也是用 template 字段定义的。在这个字段中，定义了一个使用 fluentd-elasticsearch:1.20 镜像的容器，而且这个容器挂载了两个 hostPath 类型的 Volume，分别对应宿主机的 /var/log 目录和 /var/lib/docker/containers 目录。

fluentd 启动之后，它会从这两个目录里搜集日志信息，并转发给 ElasticSearch 保存。这样，我们通过 ElasticSearch 就可以很方便地检索这些日志了。

**DaemonSet 又是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？**

DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 name=fluentd-elasticsearch 标签的 Pod 在运行。

而检查的结果，可能有这么三种情况：

1. 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；

2. 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；

3. 正好只有一个这种 Pod，那说明这个节点是正常的。

**如何在指定的 Node 上创建新 Pod 呢？**

通过下面的例子可以看到 nodeAffinity 字段进行控制。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```

在这个 Pod 里，我声明了一个 spec.affinity 字段，然后定义了一个 nodeAffinity。其中，spec.affinity 字段，是 Pod 里跟调度相关的一个字段。定义的 nodeAffinity 的含义是：

* requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个 nodeAffinity 必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个 nodeAffinity；

* 这个 Pod，将来只允许运行在“metadata.name” 是“node-geektime”的节点上。

所以，我们的 DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 nodeAffinity 定义。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。当然，DaemonSet 并不需要修改用户提交的 YAML 文件里的 Pod 模板，而是在向 Kubernetes 发起请求之前，直接修改根据模板生成的 Pod 对象。

此外，DaemonSet 还会给这个 Pod 自动加上另外一个与调度相关的字段，叫作 tolerations。这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）。而 DaemonSet 自动加上的 tolerations 字段，格式如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

这个 Toleration 的含义是：“容忍”所有被标记为 unschedulable“污点”的 Node；“容忍”的效果是允许调度。而在正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的 Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功。

假如当前 DaemonSet 管理的，是一个网络插件的 Agent Pod，那么你就必须在这个 DaemonSet 的 YAML 文件里，给它的 Pod 模板加上一个能够“容忍”node.kubernetes.io/network-unavailable“污点”的 Toleration。正如下面这个例子所示：

```yaml
...
template:
    metadata:
      labels:
        name: network-plugin-agent
    spec:
      tolerations:
      - key: node.kubernetes.io/network-unavailable
        operator: Exists
        effect: NoSchedule
```

在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为node.kubernetes.io/network-unavailable的“污点”。而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。

这种机制，正是我们在部署 Kubernetes 集群的时候，能够先部署 Kubernetes 本身、再部署网络插件的根本原因：因为当时我们所创建的 Weave 的 YAML，实际上就是一个 DaemonSet。

至此，通过上面这些内容，你应该能够明白，DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。

在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个 Toleration，从而忽略节点的 unschedulable“污点”。

当然，你也可以在 Pod 模板里加上更多种类的 Toleration，从而利用 DaemonSet 实现自己的目的。比如，在这个 fluentd-elasticsearch DaemonSet 里，我就给它加上了这样的 Toleration：

```yaml
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
```

这是因为在默认情况下，Kubernetes 集群不允许用户在 Master 节点部署 Pod。因为，Master 节点默认携带了一个叫作node-role.kubernetes.io/master的“污点”。所以，为了能在 Master 节点上部署 DaemonSet 的 Pod，我就必须让这个 Pod“容忍”这个“污点”。

### 应用举例说明

```shell
# kubectl create -f fluentd-elasticsearch.yaml
```

在 DaemonSet 上，我们一般都应该加上 resources 字段，来限制它的 CPU 和内存使用，防止它占用过多的宿主机资源。

而创建成功后，你就能看到，如果有 N 个节点，就会有 N 个 fluentd-elasticsearch Pod 在运行。比如在我们的例子里，会有一个 Pod，如下所示：

```shell
# kubectl get pod -n kube-system -l name=fluentd-elasticsearch
NAME                          READY     STATUS    RESTARTS   AGE
fluentd-elasticsearch-lfd4c   1/1       Running   0          43s
```

而如果你此时通过 kubectl get 查看一下 Kubernetes 集群里的 DaemonSet 对象：

```shell
# kubectl -n kube-system get ds fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   1         1         1         1            1           <none>          4m
```

就会发现 DaemonSet 和 Deployment 一样，也有 DESIRED、CURRENT 等多个状态字段。这也就意味着，DaemonSet 可以像 Deployment 那样，进行版本管理。这个版本，可以使用 kubectl rollout history 看到：

```shell
# kubectl rollout history daemonset fluentd-elasticsearch -n kube-system 
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
```

我们来把这个 DaemonSet 的容器镜像版本到 v2.2.0：

```shell
# kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=gcr.azk8s.cn/google-containers/fluentd-elasticsearch:v2.2.0 --record -n=kube-system
```

这个 kubectl set image 命令里，第一个 fluentd-elasticsearch 是 DaemonSet 的名字，第二个 fluentd-elasticsearch 是容器的名字。

下面使通过`kubectl rollout status`命令进行查看更新的过程：

```shell
# kubectl rollout status ds/fluentd-elasticsearch -n kube-system
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 1 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 of 1 updated pods are available...
daemon set "fluentd-elasticsearch" successfully rolled out
```

通过在 DaemonSet 更新命令添加 `--record` 参数，这样就可以像 Deployment 一样，将 DaemonSet 回滚到某个指定的历史版本。

```shell
# kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=gcr.azk8s.cn/google-containers/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
```

由于Deployment 管理这些版本，靠的是“一个版本对应一个 ReplicaSet 对象”。可是，DaemonSet 控制器操作的直接就是 Pod，不可能有 ReplicaSet 这样的对象参与其中。那么，它的这些版本又是如何维护的呢？

在 Kubernetes 项目中，任何你觉得需要记录下来的状态，都可以被用 API 对象的方式实现。当然，“版本”也不例外。 Kubernetes v1.7 之后添加了一个 API 对象，名叫ControllerRevision，专门用来记录某种 Controller 对象的版本。比如，你可以通过如下命令查看 fluentd-elasticsearch 对应的ControllerRevision：

```shell
# kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-7dcf4d6fb8   daemonset.apps/fluentd-elasticsearch   2          16m
fluentd-elasticsearch-85cc9d6cd5   daemonset.apps/fluentd-elasticsearch   1          19m
# kubectl describe controllerrevision fluentd-elasticsearch-7dcf4d6fb8 -n kube-system
Name:         fluentd-elasticsearch-7dcf4d6fb8
Namespace:    kube-system
Labels:       controller-revision-hash=3879082964
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation=2
              kubernetes.io/change-cause=kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=gcr.azk8s.cn/google-containers/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
API Version:  apps/v1
Data:
  Spec:
    Template:
      $ Patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              gcr.azk8s.cn/google-containers/fluentd-elasticsearch:v2.2.0
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
          Resources:
            Limits:
              Memory:  200Mi
            Requests:
              Cpu:                     100m
              Memory:                  200Mi
          Termination Message Path:    /dev/termination-log
          Termination Message Policy:  File
          Volume Mounts:
            Mount Path:  /var/log
            Name:        varlog
            Mount Path:  /var/lib/docker/containers
            Name:        varlibdockercontainers
            Read Only:   true
        Dns Policy:      ClusterFirst
        Restart Policy:  Always
        Scheduler Name:  default-scheduler
        Security Context:
        Termination Grace Period Seconds:  30
        Tolerations:
          Effect:  NoSchedule
          Key:     node-role.kubernetes.io/master
        Volumes:
          Host Path:
            Path:  /var/log
            Type:  
          Name:    varlog
          Host Path:
            Path:  /var/lib/docker/containers
            Type:  
          Name:    varlibdockercontainers
Kind:              ControllerRevision
Metadata:
  Creation Timestamp:  2019-09-04T06:03:27Z
  Owner References:
    API Version:           apps/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  DaemonSet
    Name:                  fluentd-elasticsearch
    UID:                   4c8af4fd-ced9-11e9-9e9e-000c29c9ea98
  Resource Version:        214066
  Self Link:               /apis/apps/v1/namespaces/kube-system/controllerrevisions/fluentd-elasticsearch-7dcf4d6fb8
  UID:                     b67f504e-ced9-11e9-9e9e-000c29c9ea98
Revision:                  2
Events:                    <none>
```

这个 ControllerRevision 对象，实际上是在 Data 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 Annotation 字段保存了创建这个对象所使用的 kubectl 命令。

我们可以尝试将这个 DaemonSet 回滚到 Revision=1 时的状态：

```shell
# kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.extensions/fluentd-elasticsearch rolled back
# kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
2         kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=gcr.azk8s.cn/google-containers/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
3         <none>
# kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-7dcf4d6fb8   daemonset.apps/fluentd-elasticsearch   2          24m
fluentd-elasticsearch-85cc9d6cd5   daemonset.apps/fluentd-elasticsearch   3          27m
```

这个 kubectl rollout undo 操作，实际上相当于读取到了 Revision=1 的 ControllerRevision 对象保存的 Data 字段。而这个 Data 字段里保存的信息，就是 Revision=1 时这个 DaemonSet 的完整 API 对象。

所以，现在 DaemonSet Controller 就可以使用这个历史 API 对象，对现有的 DaemonSet 做一次 PATCH 操作（等价于执行一次 kubectl apply -f “旧的 DaemonSet 对象”），从而把这个 DaemonSet“更新”到一个旧版本。

这也是为什么，在执行完这次回滚完成后，你会发现，DaemonSet 的 Revision 并不会从 Revision 并不会从 Revision=2 退回到 1，而是会增加成 Revision=3。这是因为，一个新的 ControllerRevision 被创建了出来。











































