## 安装搭建

### docker安装

#### 关闭 `selinux`

```shell
setenforce 0
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux
```

#### 时间同步

```shell
# /usr/sbin/ntpdate 1.centos.pool.ntp.org &>/dev/null
```

#### 关闭防火墙

```shell
# systemctl stop firewalld
# systemctl disable firewalld
```

#### 关闭 `swap`

```shell
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
```

#### `docker` 安装配置

```shell
# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# yum install -y docker-ce-18.06.3.ce
# mkdir /etc/docker
# cat >/etc/docker/daemon.json<<EOF
{
  "registry-mirrors": ["https://2980aqtg.mirror.aliyuncs.com"],
  "live-restore": true
}
EOF
# systemctl start docker
# systemctl enable docker
```

#### 配置内核参数

```shell
sysctl -w net.bridge.bridge-nf-call-ip6tables=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -p
```

### 安装kubernetes

#### 设置国内 `yum` 源

```shell
cat  >/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 安装 `kubeadm`、`kubectl`、`kubelet`

```shell
# yum install -y kubelet-1.11.10 kubeadm-1.11.10 kubectl-1.11.10 kubernetes-cni
# systemctl enable kubelet
# systemctl start kubelet
```

#### 拉取镜像

```shell
docker pull gcr.azk8s.cn/google-containers/kube-apiserver:v1.11.10
docker pull gcr.azk8s.cn/google-containers/kube-controller-manager:v1.11.10
docker pull gcr.azk8s.cn/google-containers/kube-scheduler:v1.11.10
docker pull gcr.azk8s.cn/google-containers/kube-proxy:v1.11.10
docker pull gcr.azk8s.cn/google-containers/pause:3.1
docker pull gcr.azk8s.cn/google-containers/etcd:3.2.18
docker pull gcr.azk8s.cn/google-containers/coredns:1.1.3
##############################
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-apiserver:v1.11.10` k8s.gcr.io/kube-apiserver-amd64:v1.11.10
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-controller-manager:v1.11.10` k8s.gcr.io/kube-controller-manager-amd64:v1.11.10
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-scheduler:v1.11.10` k8s.gcr.io/kube-scheduler-amd64:v1.11.10
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-proxy:v1.11.10` k8s.gcr.io/kube-proxy-amd64:v1.11.10
docker tag `docker images -q gcr.azk8s.cn/google-containers/etcd:3.2.18` k8s.gcr.io/etcd-amd64:3.2.18
docker tag `docker images -q gcr.azk8s.cn/google-containers/coredns:1.1.3` k8s.gcr.io/coredns:1.1.3
docker tag `docker images -q gcr.azk8s.cn/google-containers/pause:3.1` k8s.gcr.io/pause:3.1
```

#### 初始化

为kubeadm提供一个yaml文件：

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
controllerManagerExtraArgs:
  horizontal-pod-autoscaler-use-rest-clients: "true"
  horizontal-pod-autoscaler-sync-period: "10s"
  node-monitor-grace-period: "10s"
apiServerExtraArgs:
  runtime-config: "api/all=true"
kubernetesVersion: "1.11.10"
```

初始化集群：

```shell
kubeadm init  --config kubeadm.yaml
...
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
...
  kubeadm join 192.168.120.88:6443 --token ufeify.fb885ox2lwxi8q5p --discovery-token-ca-cert-hash sha256:672729e5dc79d9b08a912c92a876d1e77fbf9de99736ace3cd7aeca09e8023f3
...
```

使用集群还需执行如下命令：

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 集群使用

### 初步使用

查看当前节点的状态：

```shell
# kubectl get nodes
NAME      STATUS     ROLES     AGE       VERSION
kubeadm   NotReady   master    12m       v1.11.10
# kubectl describe node kubeadm
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
...
  Ready            False   Tue, 06 Aug 2019 16:38:35 +0800   Tue, 06 Aug 2019 16:34:15 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

通过第一条命令可以看到当前node节点的状态为 NotReady 状态，说明当前节点还未准备好。出现这种问题我们可以通过第二条命令查看当前节点详细信息，通过输出结果 `cni config uninitialized` 可以发现是网络插件未部署导致的。

```shell
# kubectl get pods
No resources found.
# kubectl get pods -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-gb5kg          0/1       Pending   0          15m
coredns-78fcdf6894-pjqph          0/1       Pending   0          15m
etcd-kubeadm                      1/1       Running   0          14m
kube-apiserver-kubeadm            1/1       Running   0          14m
kube-controller-manager-kubeadm   1/1       Running   0          14m
kube-proxy-xsg4w                  1/1       Running   0          15m
kube-scheduler-kubeadm            1/1       Running   0          14m
```

下面我们查看这个node上的pod状态，默认的namespace中由于未启用pod 所以输出是未发现。而我们指定namespace 为kube-system 就可以看到kubernetes系统自己启动pod，通过输出结果我们可以发现两个coredns的状态为Pending。

### 部署网络插件

通知执行如下的命令可以一步部署网络：

```shell
# kubectl apply -f https://git.io/weave-kube-1.6
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.extensions/weave-net created
# 创建网络容器后稍等一会之后查看pods
# kubectl get pods -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-gb5kg          1/1       Running   0          52m
coredns-78fcdf6894-pjqph          1/1       Running   0          52m
etcd-kubeadm                      1/1       Running   0          51m
kube-apiserver-kubeadm            1/1       Running   0          51m
kube-controller-manager-kubeadm   1/1       Running   0          51m
kube-proxy-xsg4w                  1/1       Running   0          52m
kube-scheduler-kubeadm            1/1       Running   0          51m
weave-net-f8wkl                   2/2       Running   0          2m
```

### 添加节点

按照前面的操作安装docker、kubeadm，然后执行下面的命令：

```shell
kubeadm join 192.168.120.88:6443 --token ufeify.fb885ox2lwxi8q5p --discovery-token-ca-cert-hash sha256:672729e5dc79d9b08a912c92a876d1e77fbf9de99736ace3cd7aeca09e8023f3
```

成功执行后可以在master节点上查看新添加的node：

```shell
# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
kubeadm   Ready     master    1h        v1.11.10
node1     Ready     <none>    22s       v1.11.10
```

### 通过 Taint/Toleration 调整 Master 执行Pod的策略

默认情况下 Master 节点是不允许运行用户 Pod 的。这种方式是依靠kubernetes的Taint/Toleration 机制。他的原理就是一旦某个节点打上 `taint` 那么所有的 `Pod` 都不能在这个node上运行。除非在声明了 `Toleration ` 才可以在这个node 上运行pod。

为节点打上taint的命令：

```shell
kubectl taint nodes node1 foo=bar:NoSchedule
```

通过命令给node打上taint之后，意味着 `taint` 只会在调度新 `pod` 时产生左右，而不会影响已经在 `node1` 上运行的 `pod`，哪怕它们没有 `Toleration `  。

如果在yaml 文件中进行声明，需要在spec部分加入 toleration字段：

```shell
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Equal"
    value: "bar"
    effect: "NoSchedule"
```

现在在我们自己搭建的环境中，可以查看到master上面已经添加了 `taints`：

```shell
# kubectl describe node kubeadm
Name:               kubeadm
Roles:              master
...
Taints:             node-role.kubernetes.io/master:NoSchedule
```

通过以下命令可以删除 taint ：

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 部署dashboard

```shell
# docker pull gcr.azk8s.cn/google-containers/kubernetes-dashboard-amd64:v1.10.1
# docker tag `docker images -q gcr.azk8s.cn/google-containers/kubernetes-dashboard-amd64:v1.10.1` k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

可以通过如下命令查看dashboard的状态：

```shell
# kubectl get pods -n kube-system |grep dashboard
kubernetes-dashboard-5dd89b9875-scp7j   1/1       Running   0          1m
```

如何访问dashboard？在1.7版本之后默认只能通过poroxy的方式在本地访问，如果要在集群外访问可以通过Ingress，或者添加一个用户并修改dashboard的yaml文件暴露出一个端口，然后通过token的方式登录。

### 容器的存储插件

容器本身的文件默认是会随着容器的删除而删除，不存在持久化功能。而实际使用中如果想要让容器内文件持久化是可以通过volume进行持久化，而这种持久化功能有个很大的缺陷就是只能在单台机器上操作不能进行集群数据的持久化。在kubernets编排中我们为了文件的持久化可以通过PV和PVC进行操作，而底层的技术有很多种选择，可以是ceph也可以是别的，下面使用的是kubernets中基于ceph开发的存储插件rook。

```shell
# wget https://github.com/rook/rook/archive/v0.8.3.tar.gz
# tar -xf v0.8.3.tar.gz
# cd rook-0.8.3/
# cd cluster/examples/kubernetes/ceph/
# kubectl apply -f operator.yaml
# kubectl apply -f cluster.yaml 
# kubectl get pods -n rook-ceph-system
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-agent-8nhm5                 1/1       Running   0          14m
rook-ceph-agent-r2x9p                 1/1       Running   0          14m
rook-ceph-operator-745f756bd8-9shns   1/1       Running   0          15m
rook-discover-6kh75                   1/1       Running   0          14m
rook-discover-8gw9k                   1/1       Running   0          14m
# kubectl get pods -n rook-ceph
NAME                                  READY     STATUS      RESTARTS   AGE
rook-ceph-mgr-a-7944d8d79b-pq7f5      1/1       Running     0          12m
rook-ceph-mon0-9rhhw                  1/1       Running     0          13m
rook-ceph-mon1-jc7qv                  1/1       Running     0          13m
rook-ceph-mon2-tb4fx                  1/1       Running     0          12m
rook-ceph-osd-id-0-847bf58cc8-lrrkn   1/1       Running     0          11m
rook-ceph-osd-id-1-5b7b5b994b-m6tsz   1/1       Running     0          11m
rook-ceph-osd-prepare-kubeadm-mc87q   0/1       Completed   0          11m
rook-ceph-osd-prepare-node1-tw4fn     0/1       Completed   0          11m
```

## 容器创建使用

### 部署nginx

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

把上面的配置写入nginx-deployment.yaml文件中。这样的一个 YAML 文件，对应到 Kubernetes 中，就是一个 API Object（API 对象）。当你为这个对象的各个字段填好值并提交给 Kubernetes之后，Kubernetes 就会负责创建出这些对象所定义的容器或者其他类型的 API 资源。

通过下面的命令可以运行声明的资源：

```shell
# kubectl create -f nginx-deployment.yaml 
# kubectl get pods -l app=nginx
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-67594d6bf6-9dm46   1/1       Running   0          47s
nginx-deployment-67594d6bf6-v4vlh   1/1       Running   0          47s
```

kubectl get 指令的作用，就是从 Kubernetes 里面获取（GET）指定的 API 对象。可以看到，在这里我还加上了一个 -l 参数，即获取所有匹配 app: nginx 标签的 Pod。需要注意的是，在命令行中，所有 key-value 格式的参数，都使用“=”而非“:”表示。

如果要查看一个API的细节可以通过如下方式：

```shell
# kubectl describe pod nginx-deployment-67594d6bf6-9dm46
Name:               nginx-deployment-67594d6bf6-9dm46
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               kubeadm/192.168.120.88
Start Time:         Wed, 07 Aug 2019 13:42:15 +0800
Labels:             app=nginx
                    pod-template-hash=2315082692
Annotations:        <none>
Status:             Running
IP:                 10.32.0.11
Controlled By:      ReplicaSet/nginx-deployment-67594d6bf6
Containers:
  nginx:
    Container ID:   docker://b2c659efdba70b2ab01933d4702f96c346200e1d3721eab79a7da4fffc8465c2
    Image:          nginx:1.7.9
    Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 07 Aug 2019 13:42:37 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-2n8nl (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-2n8nl:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-2n8nl
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  1m    default-scheduler  Successfully assigned default/nginx-deployment-67594d6bf6-9dm46 to kubeadm
  Normal  Pulling    1m    kubelet, kubeadm   pulling image "nginx:1.7.9"
  Normal  Pulled     1m    kubelet, kubeadm   Successfully pulled image "nginx:1.7.9"
  Normal  Created    1m    kubelet, kubeadm   Created container
  Normal  Started    1m    kubelet, kubeadm   Started container
```

在上面的限制的信息中我们重点关注 `Events:` 之后的信息，这个是显示该pod做了什么，其中的信息对于排错很有帮助。

### 升级nginx

前面创建集群中nginx版本为1.7.9，那下面要将版本升级到1.10.3，则需要修改nginx-deploy.yaml文件，修改部分为：

```yaml
...
    spec:
      containers:
      - name: nginx
        image: nginx:1.10.3
...
```

然后重新部署应用：

```shell
# kubectl apply -f nginx-deployment.yaml
# 通过下面的命令可以查看到升级的过程
# kubectl rollout status deployment nginx-deployment
# kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-544448dc54-lqw5g   1/1       Running   0          1m
nginx-deployment-544448dc54-ltgdv   1/1       Running   0          1m
# kubectl describe pod nginx-deployment-544448dc54-lqw5g
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  3m    default-scheduler  Successfully assigned default/nginx-deployment-544448dc54-lqw5g to node1
  Normal  Pulling    3m    kubelet, node1     pulling image "nginx:1.10.3"
  Normal  Pulled     2m    kubelet, node1     Successfully pulled image "nginx:1.10.3"
  Normal  Created    2m    kubelet, node1     Created container
  Normal  Started    2m    kubelet, node1     Started container
```

### 添加 volume

修改nginx-deployment.yaml文件如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nginx-vol
      volumes:
      - name: nginx-vol
        emptyDir: {}
```

上面例子中添加一个volumes的字段，其类型是emptyDir。该类型是不显式声明宿主机目录的 Volume。所以，Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的Volume 目录上。

```shell
# kubectl apply -f nginx-deployment.yaml
# kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-7cdcdb8648-w2nlw   1/1       Running   0          4m
nginx-deployment-7cdcdb8648-xcmwc   1/1       Running   0          4m
# docker ps |grep k8s_nginx_nginx-deployment-7cdcdb8648-w2nlw
749caa8f8dfd        0346349a1a64           "nginx -g 'daemon of…"   About an hour ago   Up About an hour                        k8s_nginx_nginx-deployment-7cdcdb8648-w2nlw_default_5ddb9620-b8e2-11e9-a8ed-000c2999e480_0
# docker inspect  749caa8f8dfd
...
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/5ddb9620-b8e2-11e9-a8ed-000c2999e480/volumes/kubernetes.io~empty-dir/nginx-vol",
                "Destination": "/usr/share/nginx/html",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
...
# echo "kubernetes-nginx-vol">/var/lib/kubelet/pods/5ddb9620-b8e2-11e9-a8ed-000c2999e480/volumes/kubernetes.io~empty-dir/nginx-vol/index.html
# 容器的IP地址自己根据describe获取
# curl 10.32.0.11  
kubernetes-nginx-vol
```

Kubernetes 也提供了显式的 Volume 定义，它叫做 hostPath。比如下面的这个 YAML文件：

```yaml
 ...   
    volumes:
      - name: nginx-vol
        hostPath: 
          path: /var/data
```

按照上面的配置升级配置：

```shell
# kubectl apply -f nginx-deployment.yaml
# kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-69bd8b4999-g87wn   1/1       Running   0          11s
nginx-deployment-69bd8b4999-pqwxz   1/1       Running   0          10s
# echo "kubernetes-nginx-vol-hostPath">/var/data/index.html
# curl 10.32.0.12
kubernetes-nginx-vol-hostPath
```

同时也可以通过exec登录到pod中进行查看：

```shell
# kubectl exec -it nginx-deployment-69bd8b4999-g87wn  -- bash
root@nginx-deployment-69bd8b4999-g87wn:/# cat /usr/share/nginx/html/index.html 
kubernetes-nginx-vol-hostPath
```








