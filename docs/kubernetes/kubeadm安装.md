## `kubeadm`
### 介绍
`Kubeadm` 是一个工具，它提供了 `kubeadm init` 以及 `kubeadm join` 这两个命令作为快速创建 `kubernetes` 集群的最佳实践。

### 安装前工作
#### 关闭 `selinux`
```shell
# setenforce 0
# sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux
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
#### ssh免密登录
这是在 192.168.122.100 主机上生成公私钥，然后分发到集群中其他主机中去。
```shell
# ssh-keygen
# ssh-copy-id 192.168.122.101
```
#### 主机名解析
三台机器都要添加各主机名的解析信息。
```shell
# tail -2 /etc/hosts
192.168.122.100  kubeadm
192.168.122.101  node1
```
#### `docker` 安装
```shell
# yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# yum install -y docker-ce-18.09.8
```
两台机器均需要安装docker。
#### 配置转发参数
```shell
cat  >/etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p
```
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
#### 安装 `docker`
```shell
# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
# yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# yum install -y docker-ce-18.09.8
# systemctl start docker
# systemctl enable docker
```
#### 安装 `kubeadm`、`kubectl`、`kubelet`
```shell
# yum install -y kubelet-1.14.1 kubeadm-1.14.1 kubectl-1.14.1 kubernetes-cni
# systemctl enable kubelet
# systemctl start kubelet
```
### 安装配置 `master` 节点
#### `master` 节点拉取镜像
下载镜像
```shell
docker pull gcr.azk8s.cn/google-containers/kube-apiserver:v1.14.1
docker pull gcr.azk8s.cn/google-containers/kube-controller-manager:v1.14.1
docker pull gcr.azk8s.cn/google-containers/kube-scheduler:v1.14.1
docker pull gcr.azk8s.cn/google-containers/kube-proxy:v1.14.1
docker pull gcr.azk8s.cn/google-containers/pause:3.1
docker pull gcr.azk8s.cn/google-containers/etcd:3.3.10
docker pull gcr.azk8s.cn/google-containers/coredns:1.3.1
```
修改镜像名称
```shell
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-apiserver:v1.14.1` k8s.gcr.io/kube-apiserver:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-controller-manager:v1.14.1` k8s.gcr.io/kube-controller-manager:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-scheduler:v1.14.1` k8s.gcr.io/kube-scheduler:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-proxy:v1.14.1` k8s.gcr.io/kube-proxy:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/pause:3.1` k8s.gcr.io/pause:3.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/etcd:3.3.10` k8s.gcr.io/etcd:3.3.10
docker tag `docker images -q gcr.azk8s.cn/google-containers/coredns:1.3.1` k8s.gcr.io/coredns:1.3.1
```
#### 初始化
```shell
# kubeadm init --kubernetes-version=v1.14.1 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.122.100
[init] Using Kubernetes version: v1.14.1
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubeadm localhost] and IPs [192.168.122.100 127.0.0.1 ::1]
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubeadm localhost] and IPs [192.168.122.100 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubeadm kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.122.100]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 26.011198 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node kubeadm as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubeadm as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 3agn33.ck5d619l9nztsuhp
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.122.100:6443 --token 3agn33.ck5d619l9nztsuhp \
    --discovery-token-ca-cert-hash sha256:a9846be19a084e7c60a97a6cc9aff5a7d1cef276e41e07cbfbbd05a6bc03bafd
```
#### 查看集群信息
```shell
# mkdir -p $HOME/.kube
# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
# kubectl get nodes
NAME      STATUS     ROLES    AGE   VERSION
kubeadm   NotReady   master   16m   v1.14.1
# kubectl get pods --all-namespaces
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-fb8b8dccf-pf5lm           0/1     Pending   0          19m
kube-system   coredns-fb8b8dccf-xznsb           0/1     Pending   0          19m
kube-system   etcd-kubeadm                      1/1     Running   0          18m
kube-system   kube-apiserver-kubeadm            1/1     Running   0          18m
kube-system   kube-controller-manager-kubeadm   1/1     Running   0          18m
kube-system   kube-proxy-98nm9                  1/1     Running   0          19m
kube-system   kube-scheduler-kubeadm            1/1     Running   0          18m
```
#### `flannel` 网络配置
``` shell
# wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
# kubectl apply -f kube-flannel.yml
```
`flannel` 默认会使用主机的第一张网卡，如果你有多张网卡，需要通过配置单独指定。修改 `kube-flannel.yml` 中的以下部分

```shell
......
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth0  # 指定内网网卡
......
```
可以看到 1.12 版本的 `kubeadm` 额外给 `node1` 节点设置了一个污点(Taint)：`node.kubernetes.io/not-ready:NoSchedule`，很容易理解，即如果节点还没有 `ready` 之前，是不接受调度的。可是如果 `Kubernetes` 的网络插件还没有部署的话，节点是不会进入 `ready` 状态的。因此我们修改以下 `kube-flannel.yaml` 的内容，加入对 `node.kubernetes.io/not-ready:NoSchedule` 这个污点的容忍：
```shell
......
tolerations:
- key: node-role.kubernetes.io/master
  operator: Exists
  effect: NoSchedule
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoSchedule
......
```
重新 `apply` 一下 `kubectl apply -f kube-flannel.yml`:
```shell
# kubectl apply -f kube-flannel.yml
```
#### 验证集群
```shell
# kubectl get nodes
NAME      STATUS   ROLES    AGE   VERSION
kubeadm   Ready    master   62m   v1.14.1
# kubectl get pod --all-namespaces
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-fb8b8dccf-pf5lm           1/1     Running   0          63m
kube-system   coredns-fb8b8dccf-xznsb           1/1     Running   0          63m
kube-system   etcd-kubeadm                      1/1     Running   0          63m
kube-system   kube-apiserver-kubeadm            1/1     Running   0          62m
kube-system   kube-controller-manager-kubeadm   1/1     Running   0          63m
kube-system   kube-flannel-ds-jvltn             1/1     Running   0          2m33s
kube-system   kube-proxy-98nm9                  1/1     Running   0          63m
kube-system   kube-scheduler-kubeadm            1/1     Running   0          63m
# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE     IP                NODE      NOMINATED NODE   READINESS GATES
kube-system   coredns-fb8b8dccf-pf5lm           1/1     Running   0          64m     10.244.0.3        kubeadm   <none>           <none>
kube-system   coredns-fb8b8dccf-xznsb           1/1     Running   0          64m     10.244.0.2        kubeadm   <none>           <none>
kube-system   etcd-kubeadm                      1/1     Running   0          63m     192.168.122.100   kubeadm   <none>           <none>
kube-system   kube-apiserver-kubeadm            1/1     Running   0          63m     192.168.122.100   kubeadm   <none>           <none>
kube-system   kube-controller-manager-kubeadm   1/1     Running   0          63m     192.168.122.100   kubeadm   <none>           <none>
kube-system   kube-flannel-ds-jvltn             1/1     Running   0          2m55s   192.168.122.100   kubeadm   <none>           <none>
kube-system   kube-proxy-98nm9                  1/1     Running   0          64m     192.168.122.100   kubeadm   <none>           <none>
kube-system   kube-scheduler-kubeadm            1/1     Running   0          63m     192.168.122.100   kubeadm   <none>           <none>
```
### `Node` 节点加入集群
#### 镜像拉取
`Node` 节点加入到集群之前需要进行前面的 **安装前工作** 方面的操作，之后才是下面的操作：
```shell
docker pull gcr.azk8s.cn/google-containers/kube-apiserver:v1.14.1
docker pull gcr.azk8s.cn/google-containers/kube-controller-manager:v1.14.1
docker pull gcr.azk8s.cn/google-containers/kube-scheduler:v1.14.1
docker pull gcr.azk8s.cn/google-containers/kube-proxy:v1.14.1
docker pull gcr.azk8s.cn/google-containers/pause:3.1
docker pull gcr.azk8s.cn/google-containers/etcd:3.3.10
docker pull gcr.azk8s.cn/google-containers/coredns:1.3.1
```
```shell
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-apiserver:v1.14.1` k8s.gcr.io/kube-apiserver:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-controller-manager:v1.14.1` k8s.gcr.io/kube-controller-manager:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-scheduler:v1.14.1` k8s.gcr.io/kube-scheduler:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/kube-proxy:v1.14.1` k8s.gcr.io/kube-proxy:v1.14.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/pause:3.1` k8s.gcr.io/pause:3.1
docker tag `docker images -q gcr.azk8s.cn/google-containers/etcd:3.3.10` k8s.gcr.io/etcd:3.3.10
docker tag `docker images -q gcr.azk8s.cn/google-containers/coredns:1.3.1` k8s.gcr.io/coredns:1.3.1
```
#### 加入集群
```shell
# kubeadm join 192.168.122.100:6443 --token 3agn33.ck5d619l9nztsuhp --discovery-token-ca-cert-hash sha256:a9846be19a084e7c60a97a6cc9aff5a7d1cef276e41e07cbfbbd05a6bc03bafd
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
#### 拷贝集群认证文件
```shell
mkdir -p $HOME/.kube
scp root@192.168.122.100:/etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
#### 查看验证集群
```shell
# kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
kubeadm   Ready    master   106m    v1.14.1
node1     Ready    <none>   5m13s   v1.14.1
```
#### 移除集群 `Node`
例如：如果要在集群中移除 `node2` 的机器，则需要先在 `master` 节点做如下操作：
```shell
kubectl drain node2 --delete-local-data --force --ignore-daemonsets
kubectl delete node node2
```
在 `node2` 节点做如下操作：
```shell
kubeadm reset
ifconfig flannel.1 down
ip link delete flannel.1
ifconfig cni0 down  
ip link delete cni0
rm -rf /var/lib/cni/
```
在 `node1` 节点做如下操作：
```shell
kubectl delete node node2
```

## dashboard 安装

### 镜像下载

```shell
[root@kubeadm ~]# docker pull gcr.azk8s.cn/google-containers/kubernetes-dashboard-amd64:v1.10.1
[root@kubeadm ~]# docker tag `docker images -q gcr.azk8s.cn/google-containers/kubernetes-dashboard-amd64:v1.10.1` k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
```

### 配置文件修改

默认配置文件中修改最后一段配置将主机的30001端口暴露出去提供访问。

```shell
# wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
# vim kubernetes-dashboard.yaml
.....
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
# kubectl apply -f kubernetes-dashboard.yaml 
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```

### 创建 admin 用户

注意 `Tokens:              admin-token-chmpl` 这个字段每次生成的都不一样。

```shell
# cat admin-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
# kubectl apply -f admin-user.yaml 
# kubectl describe serviceaccount admin -n kube-system
Name:                admin
Namespace:           kube-system
Labels:              k8s-app=kubernetes-dashboard
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"admin","namesp...
Image pull secrets:  <none>
Mountable secrets:   admin-token-chmpl
Tokens:              admin-token-chmpl
Events:              <none>
[root@kubeadm ~]# kubectl describe secret admin-token-chmpl  -n kube-system                 
Name:         admin-token-chmpl
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 4269d40a-b2b7-11e9-b3b8-000c29f5caa4

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1jaG1wbCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjQyNjlkNDBhLWIyYjctMTFlOS1iM2I4LTAwMGMyOWY1Y2FhNCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.cUOynRY-HvKpJtHnvAv536DPmExP-5EAgPPBWT7o_Wr415wuqFpgh1Uii2h5ZMMZPqe8x179sgbqGBvEh_1dyJz-xn-xPMxCeNkmUp-CV1caGUSlr6ESgsqzTCxy5PKTmF5T4zzDm3wByLetV4BiGIlMRAiWOGyQWWs_sOKqEYlh66P1sJxfoGKG_-xpwVgECf4L8McrlTkt0kYk-oN7d_F-efdsUDWqcT7MlFvr7Tn2TIqtzz9NLdJv1UnO8CE9rYPzMkUhg-GWyxYW86WQM-_kApHhHqP70SAips8Llc1NJ3Zwz_cfAg65DCKZYkhMWzkCabK8-ESMZ92IIF-iRA
ca.crt:     1025 bytes
```

### 登录 dashboard 

token 是前面查看所获得的字段信息，在火狐浏览器输入 `https://192.168.120.71:30001` ，通过token 导入便可登录。

![](img/login.png)

### 查看集群信息

```shell
# kubectl get pods -n kube-system
NAME                                    READY   STATUS              RESTARTS   AGE
coredns-fb8b8dccf-8q5hh                 1/1     Running             0          7h39m
coredns-fb8b8dccf-vvqxh                 1/1     Running             0          7h39m
etcd-kubeadm                            1/1     Running             0          7h39m
kube-apiserver-kubeadm                  1/1     Running             0          7h39m
kube-controller-manager-kubeadm         1/1     Running             0          7h38m
kube-flannel-ds-llzxl                   1/1     Running             0          7h30m
kube-flannel-ds-s9ftf                   1/1     Running             0          7h25m
kube-proxy-k5npj                        1/1     Running             0          7h25m
kube-proxy-w647c                        1/1     Running             0          7h39m
kube-scheduler-kubeadm                  1/1     Running             0          7h39m
kubernetes-dashboard-5f7b999d65-66qcd   0/1     ContainerCreating   0          7s
# kubectl describe svc kubernetes-dashboard -n kube-system
# kubectl get pods -n kube-system -o wide
# kubectl get service -n kube-system -o wide
```



<https://mritd.me/2018/04/19/set-up-kubernetes-1.10.1-cluster-by-hyperkube/>

