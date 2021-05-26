## 声明式 API

### 介绍

前面学习介绍了很多Kubernetes的API对象，这些 API 对象，有的是用来描述应用，有的则是为应用提供各种各样的服务。但是，无一例外地，为了使用这些 API 对象提供的能力，你都需要编写一个对应的 YAML 文件交给 Kubernetes。

这个 YAML 文件，正是 Kubernetes 声明式 API 所必须具备的一个要素。不过，是不是只要用 YAML 文件代替了命令行操作，就是声明式 API 了呢？

下面在本地编写一个 Deployment 的 YAML 文件：

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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

然后，我们还需要使用 kubectl create 命令在 Kubernetes 里创建这个 Deployment 对象：

```shell
# kubectl create -f nginx-deplyment.yaml 
deployment.apps/nginx-deployment created
```

我们前面曾经使用过 kubectl set image 和 kubectl edit 命令，来直接修改 Kubernetes 里的 API 对象。那么通过修改YAML文件也能达到相同的效果。我们可以修改这个 YAML 文件里的 Pod 模板部分，把 Nginx 容器的镜像改成 1.16.0：

```yaml
...
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.0
```

而接下来，我们就可以执行一句 kubectl replace 操作，来完成这个 Deployment 的更新：

```shell
# kubectl replace -f nginx-deplyment.yaml 
deployment.apps/nginx-deployment replaced
```

对于上面这种先 kubectl create，再 replace 的操作，我们称为命令式配置文件操作。并不是声明式API。而实际如果要使用声明式API，则需要通过kubectl apply 命令。

现在，我就使用 kubectl apply 命令来创建这个 Deployment（创建之前需要将前面创建的给删除掉）：

```shell
# kubectl apply -f nginx-deplyment.yaml
deployment.apps/nginx-deployment created
```

这样，Nginx 的 Deployment 就被创建了出来，这看起来跟 kubectl create 的效果一样。 然后再修改一下 nginx-deplyment.yaml  里定义的镜像：

```yaml
...
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
```

在修改完这个 YAML 文件之后，我不再使用 kubectl replace 命令进行更新，而是继续执行一条 kubectl apply 命令，即：

```shell
# kubectl apply -f nginx-deplyment.yaml
deployment.apps/nginx-deployment configured
```

这时，Kubernetes 就会立即触发这个 Deployment 的“滚动更新”。

它跟 kubectl replace 命令有什么本质区别吗？

kubectl replace 的执行过程，是使用新的 YAML 文件中的 API 对象，替换原有的 API 对象；而 kubectl apply，则是执行了一个对原有 API 对象的 PATCH 操作。kubectl set image 和 kubectl edit 也是对已有 API 对象的修改。

更进一步地，这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。

### 总结

1. 所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。

2. “声明式 API”允许有多个 API 写端，以 PATCH  的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容。

3. 有了上述两个能力，Kubernetes 项目才可以基于对 API 对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。





