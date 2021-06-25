## 优化 Ansible 速度

### 开启 SSH 长连接

如果 Ansible 中控机的 `ssh -V` 版本高于5.6时，可以直接在 ansible.cfg 文件中设置 SSH 长连接即可。

```
ssh_args = -C -o ControlMaster=auto -o ControlPersist=6h
control_path_dir = ~/.ansible/cp
```

### 开启 pipelining

原先ansible 执行流程就是把生成好的本地pyhon 脚本 PUT 到远端服务器。如果开启了pipelining，这个过程将在 SSH 会话中进行，这样会提升执行速度。

```\
pipelining = True
```

### 开启 accelerate

```shell
[accelerate]
accelerate_port = 5099
accelerate_timeout = 30
accelerate_connect_timeout = 5.0
```

### 设置 facts 缓存

```shel
gathering = smart
fact_caching_timeout = 86400
fact_caching = jsonfile
fact_caching_connection = /dev/shm/ansible_fact_cache
```

## 统一目录结构

```shell
project/
├── filter_plugins          # 自定义 filter 插件存放目录
├── fooapp                  # Fooapp 片色目录 ( 与 common 角色目录平级)
├── group_vars             
│   ├── group1              # group1 自定义变量文件
│   └── group2              # group2 自定义变量文件
├── host_vars
│   ├── hostname1           # hostname1 自定义变量文件
│   └── hostname2           # hostname1 自定义变量文件
├── library                 # 自定义模块存放目录
├── monitoring              # Monitoring 角色目录 ( 与 common 角色目录平级)
├── roles                   # Role 存放目录
│   └── common              # common 角色目录
│       ├── defaults       
│       │   └── main.yml    # common 角色自定义文件 (优先级低)
│       ├── files
│       │   ├── bar.txt     # common 角色 files 资源文件
│       │   └── foo.sh      # common 角色 files 资源文件
│       ├── handlers
│       │   └── main.yml    # common 角色 handlers 入口文件
│       ├── meta
│       │   └── main.yml    # common 角色 依赖文件
│       ├── tasks
│       │   └── main.yml    # common 角色 task 入口文件
│       ├── template
│       │   └── ntp.conf.j2 # common 角色 template 文件
│       └── vars
│           └── main.yml    # common 角色 变量定义文件
├── site.yaml               # Playbook 统一入口文件
├── stage                   # stage 环境的 inventory 文件
├── webservers.yml          # 特殊 Playbook 文件
└── webtier                 # webtier 角色目录 ( 与 common 角色目录平级)
```

