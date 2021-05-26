## 持久化存储

### PersistentVolume（PV）

PersistentVolume 就是持久化存储数据卷。这个 API 对象主要定义的是一个持久化存储在宿主机上的目录，比如一个 NFS 的挂载目录。通常情况下，PV 对象是由运维人员事先创建在 Kubernetes 集群里待用的。比如，运维人员可以定义这样一个 NFS 类型的 PV，如下所示：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
```

### PersistentVolumeClaim（PVC）

PersistentVolumeClaim 就是 Pod 所希望使用的持久化存储的属性。比如，Volume 存储的大小、可读写权限等等。PVC 对象通常由开发人员创建；或者以 PVC 模板的方式成为 StatefulSet 的一部分，然后由 StatefulSet 控制器负责创建带编号的 PVC。

比如，开发人员可以声明一个 1 GiB 大小的 PVC，如下所示：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

### PV 与 PVC 使用关系

而用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定。这里要检查的条件，包括两部分：

* 第一个条件，当然是 PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。

* 第二个条件，则是 PV 和 PVC 的 storageClassName 字段必须一样。

在成功地将 PVC 和 PV 进行绑定之后，Pod 就能够像使用 hostPath 等常规类型的 Volume 一样，在自己的 YAML 文件里声明使用这个 PVC 了，如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```

可以看到，Pod 需要做的，就是在 volumes 字段里声明自己要使用的 PVC 名字。接下来，等这个 Pod 创建之后，kubelet 就会把这个 PVC 所对应的 PV，也就是一个 NFS 类型的 Volume，挂载在这个 Pod 容器内的目录上。

PVC 和 PV 的设计，其实跟“面向对象”的思想完全一致。PVC 可以理解为持久化存储的“接口”，它提供了对某种持久化存储的描述，但不提供具体的实现；而这个持久化存储的实现部分则由 PV 负责完成。

你在创建 Pod 的时候，系统里并没有合适的 PV 跟它定义的 PVC 绑定，也就是说此时容器想要使用的 Volume 不存在。这时候，Pod 的启动就会报错。但是过了一会我们创建了一个对应的 PV。这时候，我们当然希望 Kubernetes 能够再次完成 PVC 和 PV 的绑定操作，从而启动 Pod。

所以在 Kubernetes 中，实际上存在着一个专门处理持久化存储的控制器，叫作 Volume Controller。这个 Volume Controller 维护着多个控制循环，其中有一个循环，扮演的就是撮合 PV 和 PVC 的“红娘”的角色。它的名字叫作 PersistentVolumeController。

PersistentVolumeController 会不断地查看当前每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个“单身”的 PVC 进行绑定。这样，Kubernetes 就可以保证用户提交的每一个 PVC，只要有合适的 PV 出现，它就能够很快进入绑定状态。

而所谓将一个 PV 与 PVC 进行“绑定”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 spec.volumeName 字段上。所以，接下来 Kubernetes 只要获取到这个 PVC 对象，就一定能够找到它所绑定的 PV。

### Volume 和持久化 Volume

* 所谓容器的 Volume，其实就是将一个宿主机上的目录，跟一个容器里的目录绑定挂载在了一起。

* 所谓的“持久化 Volume”，指的就是这个宿主机上的目录，具备“持久性”。即：这个目录里面的内容，既不会因为容器的删除而被清理掉，也不会跟当前的宿主机绑定。这样，当容器被重启或者在其他节点上重建出来之后，它仍然能够通过挂载这个 Volume，访问到这些内容。

显然，前面使用的 hostPath 和 emptyDir 类型的 Volume 并不具备这个特征：它们既有可能被 kubelet 清理掉，也不能被“迁移”到其他节点上。所以，大多数情况下，持久化 Volume 的实现，往往依赖于一个远程存储服务，比如：远程文件存储（比如，NFS、GlusterFS）、远程块存储（比如，公有云提供的远程磁盘）等等。

而 Kubernetes 需要做的工作，就是使用这些存储服务，来为容器准备一个持久化的宿主机目录，以供将来进行绑定挂载时使用。而所谓“持久化”，指的是容器在这个目录里写入的文件，都会保存在远程存储中，从而使得这个目录具备了“持久性”。

**这个准备“持久化”宿主机目录的过程，我们可以形象地称为“两阶段处理”。**

当一个 Pod 调度到一个节点上之后，kubelet 就要负责为这个 Pod 创建它的 Volume 目录。默认情况下，kubelet 为 Volume 创建的目录是如下所示的一个宿主机上的路径：

```
/var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >
```

接下来，kubelet 要做的操作就取决于你的 Volume 类型了。如果你的 Volume 类型是远程块存储，比如 Google Cloud 的 Persistent Disk （GCE 提供的远程磁盘服务），那么 kubelet 就需要先调用 Goolge Cloud 的 API，将它所提供的 Persistent Disk 挂载到 Pod 所在的宿主机上。相当于执行：

```shell
gcloud compute instances attach-disk < 虚拟机名字 > --disk < 远程磁盘名字 >
```

这一步为虚拟机挂载远程磁盘的操作，对应的正是“两阶段处理”的第一阶段。在 Kubernetes 中，我们把这个阶段称为 Attach。

Attach 阶段完成后，为了能够使用这个远程磁盘，kubelet 还要进行第二个操作，即：格式化这个磁盘设备，然后将它挂载到宿主机指定的挂载点上。这个挂载点就是 Volume 的宿主机目录。这一步相当于执行：

```shell
# 通过 lsblk 命令获取磁盘设备 ID
$ sudo lsblk
# 格式化成 ext4 格式
$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/< 磁盘设备 ID>
# 挂载到挂载点
$ sudo mkdir -p /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >
```

这个将磁盘设备格式化并挂载到 Volume 宿主机目录的操作，对应的正是“两阶段处理”的第二个阶段，我们一般称为：Mount。Mount 阶段完成后，这个 Volume 的宿主机目录就是一个“持久化”的目录了，容器在它里面写入的内容，会保存在 Google Cloud 的远程磁盘中。

而如果你的 Volume 类型是远程文件存储（比如 NFS）的话，kubelet 的处理过程就会更简单一些。因为在这种情况下，kubelet 可以跳过“第一阶段”（Attach）的操作，这是因为一般来说，远程文件存储并没有一个“存储设备”需要挂载在宿主机上。所以，kubelet 会直接从“第二阶段”（Mount）开始准备宿主机上的 Volume 目录。

在这一步，kubelet 需要作为 client，将远端 NFS 服务器的目录（比如：“/”目录），挂载到 Volume 的宿主机目录上，即相当于执行如下所示的命令：

```shell
mount -t nfs <NFS 服务器地址 >:/ /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 > 
```

通过这个挂载操作，Volume 的宿主机目录就成为了一个远程 NFS 目录的挂载点，后面你在这个目录里写入的所有文件，都会被保存在远程 NFS 服务器上。所以，我们也就完成了对这个 Volume 宿主机目录的“持久化”。

### Volume 的“两阶段处理” 

在具体的 Volume 插件的实现接口上，Kubernetes 分别给这两个阶段提供了两种不同的参数列表：

* 对于“第一阶段”（Attach），Kubernetes 提供的可用参数是 nodeName，即宿主机的名字。

* 而对于“第二阶段”（Mount），Kubernetes 提供的可用参数是 dir，即 Volume 的宿主机目录。

而经过了“两阶段处理”，我们就得到了一个“持久化”的 Volume 宿主机目录。所以，接下来，kubelet 只要把这个 Volume 目录通过 CRI 里的 Mounts 参数，传递给 Docker，然后就可以为 Pod 里的容器挂载这个“持久化”的 Volume 了。其实，这一步相当于执行了如下所示的命令：

```shell
docker run -v /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >:/< 容器内的目标目录 > 我的镜像 ...
```

这个 PV 的处理流程似乎跟 Pod 以及容器的启动流程没有太多的耦合，只要 kubelet 在向 Docker 发起 CRI 请求之前，确保“持久化”的宿主机目录已经处理完毕即可。

所以，在 Kubernetes 中，上述关于 PV 的“两阶段处理”流程，是靠独立于 kubelet 主控制循环（Kubelet Sync Loop）之外的两个控制循环来实现的。

其中，“第一阶段”的 Attach（以及 Dettach）操作，是由 Volume Controller 负责维护的，这个控制循环的名字叫作：AttachDetachController。而它的作用，就是不断地检查每一个 Pod 对应的 PV，和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作。

作为一个 Kubernetes 内置的控制器，Volume Controller 自然是 kube-controller-manager 的一部分。所以，AttachDetachController 也一定是运行在 Master 节点上的。当然，Attach 操作只需要调用公有云或者具体存储项目的 API，并不需要在具体的宿主机上执行操作，所以这个设计没有任何问题。

而“第二阶段”的 Mount（以及 Unmount）操作，必须发生在 Pod 对应的宿主机上，所以它必须是 kubelet 组件的一部分。这个控制循环的名字，叫作：VolumeManagerReconciler，它运行起来之后，是一个独立于 kubelet 主循环的 Goroutine。

通过这样将 Volume 的处理同 kubelet 的主循环解耦，Kubernetes 就避免了这些耗时的远程挂载操作拖慢 kubelet 的主控制循环，进而导致 Pod 的创建效率大幅下降的问题。实际上，kubelet 的一个主要设计原则，就是它的主控制循环绝对不可以被 block。

### StorageClass

前面人工管理 PV 的方式就叫作 Static Provisioning。这种人工方式的创建 PV 在并不适合在大型环境中使用。所以，Kubernetes 为我们提供了一套可以自动创建 PV 的机制，即：Dynamic Provisioning。Dynamic Provisioning 机制工作的核心，在于一个名叫 StorageClass 的 API 对象。

而 StorageClass 对象的作用，其实就是创建 PV 的模板。具体地说，StorageClass 对象会定义如下两个部分内容：

* 第一，PV 的属性。比如，存储类型、Volume 的大小等等。

* 第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。

有了这样两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass 了。然后，Kubernetes 就会调用该 StorageClass 声明的存储插件，创建出需要的 PV。

举个例子，运维人员定义一个如下所示的 StorageClass：

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: block-service
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicapool
  clusterNamespace: rook-ceph
  fstype: xfs
reclaimPolicy: Retain
```

在这个 YAML 文件中，我们定义的还是一个名叫 block-service 的 StorageClass，只不过它声明使的存储插件是Rook 项目。下面是创建这个 StorageClass ：

```shell
# kubectl create -f sc.yaml 
cephblockpool.ceph.rook.io/replicapool created
storageclass.storage.k8s.io/block-service created
```

这时候，作为应用开发者，我们只需要在 PVC 里指定要使用的 StorageClass 名字即可，如下所示：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: block-service
  resources:
    requests:
      storage: 2Gi
```

我们在这个 PVC 里添加了一个叫作 storageClassName 的字段，用于指定该 PVC 所要使用的 StorageClass 的名字是：block-service。创建上面文件的 PVC：

```shell
# kubectl create -f pvc.yaml
persistentvolumeclaim/claim1 created
```

可以看到，我们创建的 PVC 会绑定一个 Kubernete 自动创建的 PV，如下所示：

```shell
# kubectl get pvc
NAME      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
claim1    Bound     pvc-2de3a468-de9d-11e9-8214-000c29c9ea98   2Gi        RWO            block-service   29s
# kubectl describe pvc claim1
Name:          claim1
Namespace:     default
StorageClass:  block-service
Status:        Bound
Volume:        pvc-2de3a468-de9d-11e9-8214-000c29c9ea98
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed=yes
               pv.kubernetes.io/bound-by-controller=yes
               volume.beta.kubernetes.io/storage-provisioner=ceph.rook.io/block
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      2Gi
Access Modes:  RWO
Events:
  Type    Reason                 Age                From                                                                                       Message
  ----    ------                 ----               ----                                                                                       -------
  Normal  ExternalProvisioning   22s (x2 over 22s)  persistentvolume-controller                                                                waiting for a volume to be created, either by external provisioner "ceph.rook.io/block" or manually created by system administrator
  Normal  Provisioning           22s                ceph.rook.io/block_rook-ceph-operator-88fb86bc-gjtvh_089cc439-de76-11e9-88e9-de7f5a072050  External provisioner is provisioning volume for claim "default/claim1"
  Normal  ProvisioningSucceeded  20s                ceph.rook.io/block_rook-ceph-operator-88fb86bc-gjtvh_089cc439-de76-11e9-88e9-de7f5a072050  Successfully provisioned volume pvc-2de3a468-de9d-11e9-8214-000c29c9ea98
```

而且，通过查看这个自动创建的 PV 的属性，你就可以看到它跟我们在 PVC 里声明的存储的属性是一致的，如下所示：

```shell
# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM            STORAGECLASS    REASON    AGE
pvc-2de3a468-de9d-11e9-8214-000c29c9ea98   2Gi        RWO            Retain           Bound     default/claim1   block-service             1m
# kubectl describe pv pvc-2de3a468-de9d-11e9-8214-000c29c9ea98
Name:            pvc-2de3a468-de9d-11e9-8214-000c29c9ea98
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by=ceph.rook.io/block
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    block-service
Status:          Bound
Claim:           default/claim1
Reclaim Policy:  Retain
Access Modes:    RWO
Capacity:        2Gi
Node Affinity:   <none>
Message:         
Source:
    Type:       FlexVolume (a generic volume resource that is provisioned/attached using an exec based plugin)
    Driver:     ceph.rook.io/rook-ceph
    FSType:     xfs
    SecretRef:  <nil>
    ReadOnly:   false
    Options:    map[clusterNamespace:rook-ceph dataBlockPool: image:pvc-2de3a468-de9d-11e9-8214-000c29c9ea98 pool:replicapool storageClass:block-service]
Events:         <none>
```

此外，你还可以看到，这个自动创建出来的 PV 的 StorageClass 字段的值，也是 block-service。这是因为，Kubernetes 只会将 StorageClass 相同的 PVC 和 PV 绑定起来。

有了 Dynamic Provisioning 机制，运维人员只需要在 Kubernetes 集群里创建出数量有限的 StorageClass 对象就可以了。这就好比，运维人员在 Kubernetes 集群里创建出了各种各样的 PV 模板。这时候，当开发人员提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV。

默认支持 Dynamic Provisioning 的内置存储插件可通过下面的链接进行查看：[传送门](<https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner>)

需要注意的是，StorageClass 并不是专门为了 Dynamic Provisioning 而设计的。比如，在本篇一开始的例子里，我在 PV 和 PVC 里都声明了 storageClassName=manual。而我的集群里，实际上并没有一个名叫 manual 的 StorageClass 对象。这完全没有问题，这个时候 Kubernetes 进行的是 Static Provisioning，但在做绑定决策的时候，它依然会考虑 PV 和 PVC 的 StorageClass 定义。

而这么做的好处也很明显：这个 PVC 和 PV 的绑定关系，就完全在我自己的掌控之中。

前面使用StatefulSet 存储状态时我们并没有声明 StorageClass ，这是由于如果你的集群已经开启了名叫 DefaultStorageClass 的 Admission Plugin，它就会为 PVC 和 PV 自动添加一个默认的 StorageClass；否则，PVC 的 storageClassName 的值就是“”，这也意味着它只能够跟 storageClassName 也是“”的 PV 进行绑定。

### 总结

前面所讲这些概念之间的关系，可以用如下所示的一幅示意图描述：

![](img/pv-pvc.png)



从图中我们可以看到，在这个体系中： 

* PVC 描述的，是 Pod 想要使用的持久化存储的属性，比如存储的大小、读写权限等。

* PV 描述的，则是一个具体的 Volume 的属性，比如 Volume 的类型、挂载目录、远程存储服务器地址等。

* 而 StorageClass 的作用，则是充当 PV 的模板。并且，只有同属于一个 StorageClass 的 PV 和 PVC，才可以绑定在一起。

当然，StorageClass 的另一个重要作用，是指定 PV 的 Provisioner（存储插件）。这时候，如果你的存储插件支持 Dynamic Provisioning 的话，Kubernetes 就可以自动为你创建 PV 了。



