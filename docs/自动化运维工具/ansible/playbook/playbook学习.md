## playbook

### 构成

* Target section：   定义将要执行 playbook 的远程主机组

* Variable section： 定义 playbook 运行时需要使用的变量

* Task section：     定义将要在远程主机上执行的任务列表

* Handler section：  定义 task 执行完成以后需要调用的任务

### 第一个playbook

#### 定义主机

```shell
# vim inventory/hosts
[centos6]
192.168.120.61 ansible_ssh_pass='redhat'
192.168.120.62 ansible_ssh_pass='redhat'
192.168.120.63 ansible_ssh_pass='redhat'
```

#### 明确执行任务

远程执行用户为root

安装ntp并配置时间服务器

修改sshd登录配置文件

重启ntp和sshd这俩服务

#### 编辑playbook

```yaml
- name: CentOS 6.x initialize
  hosts: all
  vars: 
    ntp_server: ntp1.aliyun.com
  tasks:
    - name: Install ntp package 
      yum: name=ntp state=present

    - name: configure ntpd
      template: src=./ntp.conf.j2 dest=/etc/ntp.conf owner=root group=root mode=0644
      notify:
        - Restart ntpd service

    - name: configure sshd
      template: src=./sshd_config.j2 dest=/etc/ssh/sshd_config owner=root group=root mode=0644 validate='/usr/sbin/sshd -t -f %s'
      notify:
        - Restart sshd service

  handlers:
    - name: Restart ntpd service
      service: name=ntpd state=restarted

    - name: Restart sshd service
      service: name=sshd state=restarted
```

上面playbook需要说明 template 模块 validate 参数：在复制到适当位置之前要运行的验证命令。要验证的文件路径通过“%s”传递。

#### 模版文件说明

上面 ntp.conf.j2 和 sshd_config.j2 这两个模版文件和默认安装的文件有如下两个地方不同之处：

```shell
# vim ntp.conf.j2
server {{ ntp_server }} prefer
# vim sshd_config.j2
UseDNS yes
```

#### 运行前检查

检查语法

```shell
# ansible-playbook -i inventory/hosts centos6_init.yaml --syntax-check

playbook: centos6_init.yaml
```

列出要执行的主机

```shell
# ansible-playbook -i inventory/hosts centos6_init.yaml --list-hosts

playbook: centos6_init.yaml

  play #1 (all): CentOS 6.x initialize  TAGS: []
    pattern: [u'all']
    hosts (3):
      192.168.120.62
      192.168.120.63
      192.168.120.61
```

列出要执行的任务

```shell
# ansible-playbook -i inventory/hosts centos6_init.yaml --list-tasks

playbook: centos6_init.yaml

  play #1 (all): CentOS 6.x initialize  TAGS: []
    tasks:
      Install ntp package       TAGS: []
      configure ntpd    TAGS: []
      configure sshd    TAGS: []
```

#### 运行playbook

```shell
# ansible-playbook -i inventory/hosts centos6_init.yaml  
```

### 变量和引用

#### 通过 inventory 文件定义主机以及主机组变量

主机信息：

```shell
# cat inventory/hosts 
192.168.120.61 key=61
192.168.120.62 key=62
192.168.120.63 key=63
[centos6]
192.168.120.6[1:3]
[centos6:vars]
ansible_python_interpreter=/usr/bin/python2.6
```

定义如下的playbook验证变量：

```yaml
- name: display variable 
  hosts: all
  gather_facts: False
  tasks: 
    - name: display host variable from hostfile
      debug: msg="The {{ inventory_hostname }} Value is {{ key }}"
```

执行上面的playbook，可以看到前面定义的变量可以使用：

```shell
# ansible-playbook -i /root/inventory/ variable.yaml 

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is 62"
}
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is 61"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is 63"
}
......
```

上面定义的变量是在各主机上，而实际也可以在主机组中进行定义变量，如下所示：

```shell
# cat inventory/hosts 
#192.168.120.61 key=61
#192.168.120.62 key=62
#192.168.120.63 key=63
[centos6]
192.168.120.6[1:3]
[centos6:vars]
ansible_python_interpreter=/usr/bin/python2.6
key=centos6
```

#### 通过文件定义变量

通过roles 的方式指定 group_vars 和 host_vars 进行定义变量：

```shell
# tree /etc/ansible/roles/
/etc/ansible/roles/
├── group_vars
│   └── centos6
├── hosts
└── host_vars
    ├── 192.168.120.61
    ├── 192.168.120.62
    └── 192.168.120.63

2 directories, 5 files
# cat /etc/ansible/roles/group_vars/centos6 
key: CentOS6.8
# head /etc/ansible/roles/host_vars/*
==> /etc/ansible/roles/host_vars/192.168.120.61 <==
key: 192.168.120.61

==> /etc/ansible/roles/host_vars/192.168.120.62 <==
key: 192.168.120.62

==> /etc/ansible/roles/host_vars/192.168.120.63 <==
key: 192.168.120.63
# cat /etc/ansible/roles/hosts
[centos6]
192.168.120.61
192.168.120.62
192.168.120.63
```

执行命令查看变量的显示：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml 

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is 192.168.120.61"
}
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is 192.168.120.62"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is 192.168.120.63"
}
......
```

删除 host_vars 目录下定义的变量，验证 group_vars 定义的变量：

```shell
# mv host_vars /tmp/
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml 

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is CentOS6.8"
}
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is CentOS6.8"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is CentOS6.8"
}
......
```

#### 通过命令行指定

在 ansible-playbook 命令中使用 -e 选择就能指定变量：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml -e "key=KEY"

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is KEY"
}
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is KEY"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is KEY"
}
......
```

除了上面的方式还可以使用 JSON 和 YAML 文件进行文件进行传递变量：

```shell
# cat var.json 
{"key": "JSON"}
# cat var.yaml 
---
key: YAML
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml -e "@var.yaml"

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is YAML"
}
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is YAML"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is YAML"
}
......
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml -e "@var.json"

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is JSON"
}
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is JSON"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is JSON"
}
......
```

#### 在 playbook 文件中定义变量

在前面 `第一个playbook` 中使用的例子就通过指定了 ntp_server 变量进行自定义ntpserver，这种是通过vars的方式进行定义和使用。而除了改方式还可以用 vars_files 的方式进行定义。

```yaml
- name: display variable 
  hosts: all
  gather_facts: False
  vars_files:
    - var.yaml
  tasks: 
    - name: display host variable from hostfile
      debug: msg="The {{ inventory_hostname }} Value is {{ key }}"
```

var.yaml 文件为前面测试使用的问题，执行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml 

PLAY [display variable] *************************************************************************************

TASK [display host variable from hostfile] ******************************************************************
ok: [192.168.120.61] => {
    "msg": "The 192.168.120.61 Value is YAML"
}
ok: [192.168.120.62] => {
    "msg": "The 192.168.120.62 Value is YAML"
}
ok: [192.168.120.63] => {
    "msg": "The 192.168.120.63 Value is YAML"
}
......
```

#### 使用 register 内的变量

在playbook 中task 之间使可以传递数据的，比如总共有两个tasks，其中第二个task 是否执行需要判断第一个task运行后的结果。这种情况就需要在task之间进行数据传递，把第一个task执行的结果传递给第二个task ，这种task之间数据的传递是通过register 方式进行的。

```yaml
- name: display variable
  hosts: all
  gather_facts: False
  tasks:
    - name: register variable
      shell: hostname
      register: info
    - name: display variable
      debug: msg="The variable is {{ info['stdout'] }}"
```

默认 info 输出是一串字典数据，这边输出指定其中一个键可以得到如下：

```shell
 ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml  -l 192.168.120.63

PLAY [display variable] *************************************************************************************

TASK [register variable] ************************************************************************************
changed: [192.168.120.63]

TASK [display variable] *************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The variable is C6-node3"
}
......
```

#### 使用 vars_prompt 传入

如果需要通过交互式的方式给playbook传入参数，可以使用vars_prompt 进行该操作。例如：

```yaml
- name: display variable
  hosts: all
  gather_facts: False
  vars_prompt:
    - name: one
      prompt: "Please input one value:"
      private: no

    - name: two
      prompt: "Please input two value:"
      default: "Good"
      private: yes

  tasks:
    - name: display one value
      debug: msg="One value is {{ one }}"

    - name: display two value
      debug: msg="Two value is {{ two }}"
```

通过default参数可以设置默认值，而private参数则是将输入的数据是否显式的显示出来。

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/variable.yaml -l 192.168.120.63
Please input one value:: ONE
Please input two value: [Good]: 

PLAY [display variable] *************************************************************************************

TASK [display one value] ************************************************************************************
ok: [192.168.120.63] => {
    "msg": "One value is ONE"
}

TASK [display two value] ************************************************************************************
ok: [192.168.120.63] => {
    "msg": "Two value is TWO"
}
......
```

### playbook 循环

#### 标准Loops

如果重复使用某个功能，为了增加脚本的可读性，建议使用loops 的方式进行遍历使用，例子如下：

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  tasks: 
    - name: debug loops
      debug: msg="name --> {{ item }}"
      with_items:
        - one
        - two
```

运行显示结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.63             

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.63] => (item=one) => {
    "msg": "name --> one"
}
ok: [192.168.120.63] => (item=two) => {
    "msg": "name --> two"
}
......
```

with_items 的值是 python list 数据结构，可以理解为每个task会循环读取list里面的值，key的名称是item。其中list也支持python字典，如下所示：

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  tasks:
    - name: debug loops
      debug: msg="name --> {{ item.key }}  value --> {{ item.value }}"
      with_items:
        - {key: "one", value: "VALUE1"}
        - {key: "two", value: "VALUE2"}
```

运行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.63

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.63] => (item={u'key': u'one', u'value': u'VALUE1'}) => {
    "msg": "name --> one  value --> VALUE1"
}
ok: [192.168.120.63] => (item={u'key': u'two', u'value': u'VALUE2'}) => {
    "msg": "name --> two  value --> VALUE2"
}
```

#### 嵌套 Loops

嵌套loops主要实现一对多，或者多对多的合并：

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  tasks: 
    - name: debug loops
      debug: msg="name --> {{ item[0] }}  value --> {{ item[1] }}"
      with_nested:
        - ['A']
        - ['a', 'b', 'c']
```

运行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.63

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.63] => (item=[u'A', u'a']) => {
    "msg": "name --> A  value --> a"
}
ok: [192.168.120.63] => (item=[u'A', u'b']) => {
    "msg": "name --> A  value --> b"
}
ok: [192.168.120.63] => (item=[u'A', u'c']) => {
    "msg": "name --> A  value --> c"
}
......
```

#### 散列loops

散列loops相比标准loops就是变量支持更丰富的数据结构，比如：标准loops的最外层必须是Python 的list数据类型，而散列loops直接支持YAML格式的数据变量。

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  vars:
    user:
      redhat:
        name: redhat
        shell: bash
      centos:
        name: centos
        shell: zsh
        
  tasks:
    - name: debug loops
      debug: msg="name --> {{ item.key }} value --> {{ item.value.name }} shell --> {{ item.value.shell }}"
      with_dict: "{{ user }}"
```

运行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.63 

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.63] => (item={'value': {u'shell': u'zsh', u'name': u'centos'}, 'key': u'centos'}) => {
    "msg": "name --> centos value --> centos shell --> zsh"
}
ok: [192.168.120.63] => (item={'value': {u'shell': u'bash', u'name': u'redhat'}, 'key': u'redhat'}) => {
    "msg": "name --> redhat value --> redhat shell --> bash"
}
......
```

#### 文件匹配 loops

如果我们针对一个目录下指定格式的文件进行处理，可以使用 with_fileglob 循环去匹配我们需要处理的文件。

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  tasks:
    - name: debug loops
      debug: 
        msg: "files --> {{ item }}"
      with_fileglob:
        - /root/*.yaml
```

运行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.61

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.61] => (item=/root/var.yaml) => {
    "msg": "files --> /root/var.yaml"
}
ok: [192.168.120.61] => (item=/root/centos6_init.yaml) => {
    "msg": "files --> /root/centos6_init.yaml"
}
ok: [192.168.120.61] => (item=/root/loops.yaml) => {
    "msg": "files --> /root/loops.yaml"
}
ok: [192.168.120.61] => (item=/root/variable.yaml) => {
    "msg": "files --> /root/variable.yaml"
}
......
```

#### 随机loops

示例如下：

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  tasks:
    - name: debug loops
      debug:
        msg: "name --> {{ item }}"
      with_random_choice:
        - "C6-node1"
        - "C6-node2"
        - "C6-node3"
```

运行结果：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.61

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.61] => (item=C6-node3) => {
    "msg": "name --> C6-node3"
}
......
```

#### 条件判断 loops

在执行一个task之后，我们需要检测这个task 结果是否达到了预想状态，如果没有达到我们想的状态，就需要退出整个playbook执行。这个时候就需要对某个task结果一直循环检测：

```yaml
- name: display loops
  hosts: all
  gather_facts: False
  tasks:
    - name: debug loops
      shell: cat /root/ansible
      register: host
      until: host.stdout.startswith("Master")
      retreies: 5
      delay: 5
```

5 秒一次将执行 /root/ansible结果 register 给host，然后判断host.stdout.startswith的内存是否是Master字符串开头。如果条件成立此task运行完成，如果条件不成立5秒后重试，5次后还不成立则task运行失败。

```shell
# cat /root/ansible 
Master
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.61

PLAY [display loops] ****************************************************************************************

TASK [debug loops] ******************************************************************************************
FAILED - RETRYING: debug loops (5 retries left).
changed: [192.168.120.61]
......
```

#### 文件优先匹配loops

文件优先匹配会根据你传入的变量或者文件进行从上到下匹配，如果匹配到某个文件，他会用这个文件当作 {{ item }} 的值，如下所示：

```yaml
- name: display loops
  hosts: all
  gather_facts: True
  tasks:
    - name: debug loops
      debug:
        msg: "files --> {{ item }}"
      with_first_found:
        - "{{ ansible_distribution }}.yaml"
        - "var.yaml"
```

with_first_found 从上往下匹配，输出如下：

```shell
# ls var.yaml CentOS.yaml 
CentOS.yaml  var.yaml
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.61

PLAY [display loops] ****************************************************************************************

TASK [Gathering Facts] **************************************************************************************
ok: [192.168.120.61]

TASK [debug loops] ******************************************************************************************
ok: [192.168.120.61] => (item=/root/CentOS.yaml) => {
    "msg": "files --> /root/CentOS.yaml"
}
......
```

#### register loops

前面使用register用于task直接互相传递数据，一般我们会把register用在单一的task中进行临时变量存储。其实register还可以同时接受多个task的结果当作变量的临时存储：

```yaml
- name: display loops
  hosts: all
  gather_facts: True
  tasks:
    - name: debug loops
      shell: "{{ item }}"
      with_items:
        - hostname
        - uname
      register: ret

    - name: display loops
      debug:
        msg: "{% for i in ret.results %} {{ i.stdout }} {% endfor %}"
```

上面使用jinjia2的for循环把所有结果显示出来，结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/loops.yaml -l 192.168.120.61

PLAY [display loops] ****************************************************************************************

TASK [Gathering Facts] **************************************************************************************
ok: [192.168.120.61]

TASK [debug loops] ******************************************************************************************
changed: [192.168.120.61] => (item=hostname)
changed: [192.168.120.61] => (item=uname)

TASK [display loops] ****************************************************************************************
ok: [192.168.120.61] => {
    "msg": " C6-node1  Linux "
}
......
```

### playbook lookups

前面学习使用的变量都是静态定义的，而实际使用中还支持从外部数据拉取信息。例如从数据库中拉取信息给一个变量，这种使用方式就是通过 lookups 插件实现的。

#### lookups file

file 是我们经常使用的一种lookups方式，它的原理是使用Python的codecs.open打开文件然后把结果返回给变量。示例如下：

```yaml
- name: display lookups
  hosts: all
  gather_facts: False
  vars:
    contents: "{{ lookup('file', '/etc/sysconfig/network') }}"
  tasks:
    - name: debug lookups
      debug: msg="The contents is {% for i in contents.split("\n") %} {{ i }} {% endfor %}"
```

通过使用jinjia模板进行相应的格式化，运行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/lookups.yaml -l 192.168.120.61

PLAY [display lookups] **************************************************************************************

TASK [debug lookups] ****************************************************************************************
ok: [192.168.120.61] => {
    "msg": "The contents is  NETWORKING=yes  HOSTNAME=C6-node1 "
}
......
```

#### lookups password

使用 password 的方式会对传入的内容进行加密处理，示例文件如下：

```yaml
- name: display lookups
  hosts: all
  gather_facts: False
  vars:
    contents: "{{ lookup('password', 'ansible_book') }}"
  tasks:
    - name: debug lookups
      debug: 
        msg: "The contents is {{ contents }}"
```

执行结果为：

```shell
# ansible-playbook -i /etc/ansible/roles/ /root/lookups.yaml -l 192.168.120.61

PLAY [display lookups] **************************************************************************************

TASK [debug lookups] ****************************************************************************************
ok: [192.168.120.61] => {
    "msg": "The contents is S4FVS,2UhUsgOYqHEedT"
}
......
```

#### lookups pipe

示例如下：

```yaml
- name: display lookups
  hosts: all
  gather_facts: False
  vars:
    contents: "{{ lookup('pipe', 'date +%Y-%m-%d') }}"
  tasks:
    - name: debug lookups
      debug: msg="The contents is {% for i in contents.split("\n") %} {{ i }} {% endfor %}"
```

输出结果就是在控制机器上执行 `date +%Y-%m-%d`  的结果。

```shell
# ansible-playbook -i /etc/ansible/roles/ variable.yaml 

PLAY [display lookups] *************************************************************************************************************

TASK [debug lookups] ***************************************************************************************************************
ok: [192.168.120.61] => {
    "msg": "The contents is  2019-11-29 "
}
ok: [192.168.120.62] => {
    "msg": "The contents is  2019-11-29 "
}
ok: [192.168.120.63] => {
    "msg": "The contents is  2019-11-29 "
}
......
```

#### lookups template

template 在读取文件之前需要把jinjia 模板渲染完后再读取，下面指定一个 jinjia 模板文件：

```jinja2
worker_processes {{ ansible_processor_cores }};
IPaddress {{ ansible_eth0.ipv4.address }}
```

playbook 如下所示：

```yaml
- name: display lookups
  hosts: all
  gather_facts: True
  vars:
    contents: "{{ lookup('template', './lookups.j2') }}"
  tasks:
    - name: debug lookups
      debug: msg="The contents is {% for i in contents.split("\n") %} {{ i }} {% endfor %}"
```

执行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/ variable.yaml 

PLAY [display lookups] *************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [192.168.120.62]
ok: [192.168.120.63]
ok: [192.168.120.61]

TASK [debug lookups] ***************************************************************************************************************
ok: [192.168.120.61] => {
    "msg": "The contents is  worker_processes 2  IPaddress 192.168.120.61   "
}
ok: [192.168.120.62] => {
    "msg": "The contents is  worker_processes 2  IPaddress 192.168.120.62   "
}
ok: [192.168.120.63] => {
    "msg": "The contents is  worker_processes 2  IPaddress 192.168.120.63   "
}
......
```

### playbook conditionals

我们在实际应用中会碰到不同的主机执行不同的命令，或者执行某个task的时候需要做一个简单的逻辑判断，此刻就需要在写task的时候进行相对于的判断。Ansible所有conditionals方式都是使用when进行判断，when的值是一个条件表达式，如果条件成立这个task就会执行某个操作，如果条件不成立该task不执行或者某个操作会跳过。

```yaml
- name: display conditionals
  hosts: all
  tasks:
    - name: Host 192.168.120.61 run this task
      debug: 
        msg: "{{ ansible_default_ipv4.address }}"
      when: ansible_default_ipv4.address == "192.168.120.61"

    - name: memtotal < 3000M and processor_cores == 2 run this task
      debug: 
        msg: "{{ ansible_fqdn }}"
      when: ansible_memtotal_mb < 3000 and ansible_processor_cores == 2

    - name: all host run this task
      shell: hostname
      register: info

    - name: Hostname is C6-node2 Machie run this task
      debug: 
        msg: "{{ ansible_fqdn }}"
      when: info['stdout'] == 'C6-node2'

    - name: Hostname is startswith C6 run this task
      debug: 
        msg: "{{ ansible_fqdn }}"
      when: info['stdout'].startswith('C6')
```

执行结果如下：

```shell
# ansible-playbook -i /etc/ansible/roles/  variable.yaml 

PLAY [display conditionals] ********************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [192.168.120.62]
ok: [192.168.120.63]
ok: [192.168.120.61]

TASK [Host 192.168.120.61 run this task] *******************************************************************************************
ok: [192.168.120.61] => {
    "msg": "192.168.120.61"
}
skipping: [192.168.120.62]
skipping: [192.168.120.63]

TASK [memtotal < 3000M and processor_cores == 2 run this task] *********************************************************************
ok: [192.168.120.61] => {
    "msg": "C6-node1"
}
ok: [192.168.120.62] => {
    "msg": "C6-node2"
}
ok: [192.168.120.63] => {
    "msg": "C6-node3"
}

TASK [all host run this task] ******************************************************************************************************
changed: [192.168.120.62]
changed: [192.168.120.63]
changed: [192.168.120.61]

TASK [Hostname is C6-node2 Machie run this task] ***********************************************************************************
skipping: [192.168.120.61]
ok: [192.168.120.62] => {
    "msg": "C6-node2"
}
skipping: [192.168.120.63]

TASK [Hostname is startswith C6 run this task] *************************************************************************************
ok: [192.168.120.61] => {
    "msg": "C6-node1"
}
ok: [192.168.120.62] => {
    "msg": "C6-node2"
}
ok: [192.168.120.63] => {
    "msg": "C6-node3"
}
......
```

### Jinjia2 filter

ansible 和 saltstack 两个配置管理攻击都是使用jinjia2 当其默认的模板语言。下面介绍几个常用的几个filter进行学习。

```yaml
- name: display jinjia2 filter
  hosts: all
  gather_facts: False
  vars:
    list: [1, 2, 3, 4, 5]
    one: "1"
    str: "string"

  tasks:
    - name: run commands
      shell: df -h
      register: info

    - name: debug pprint filter
      debug: 
        msg: "{{ info.stdout | pprint }}"

    - name: debug conditionals filter
      debug: 
        msg: "The run commands status is changed"
      when: info is changed

    - name: debug int capitalize filter
      debug:
        msg: "The int value {{ one|int }} The lower value is {{ str | capitalize }}"

    - name: debug default filter
      debug:
        msg: "The variable value is {{ ansible | default('ansible is not define') }}"

    - name: debug list max and min filter
      debug:
        msg: "The list max value is {{ list|max }} The list min value is {{ list | min }}"

    - name: debug random filter
      debug:
        msg: "The list random value is {{ list | random }} and generate a random value is {{ 1000 |random(1,10) }}"

    - name: debug join filter
      debug:
        msg: "The join filter value is {{ list | join('+') }}"

    - name: debug replace and regex_replace filter
      debug:
        msg: "The replace value is {{ str | replace('t', 'T') }} The regex_replace value is {{ str|regex_replace('.*tr(.*)$','\\1')}}"
```

* 对 info.stdout 结果使用 pprint filter 进行格式化。
* 对info 的执行状态使用 changed filter 进行判断。
* 对 one 的值进行 int 转变，然后对str 的值进行 capitalize 格式化。
* 对ansible 变量进行判断，如果该变量定义了就引用它的值，如果没有定义就是用default的值。
* 对list 内的值进行最大值max和最小值min取值。
* 对list 内的值使用 random filter 随机挑选一个，然后随机生成1000 以内的数字，step是10 。
* 对list 内的值使用 join filter 连接一起。
* 对 str 的值使用replace 和 regex_replace 替换。

```shell
# ansible-playbook -i /etc/ansible/roles/ jinjia2.yaml  -l 192.168.120.63

PLAY [display jinjia2 filter] ******************************************************************************************************

TASK [run commands] ****************************************************************************************************************
changed: [192.168.120.63]

TASK [debug pprint filter] *********************************************************************************************************
ok: [192.168.120.63] => {
    "msg": "u'Filesystem      Size  Used Avail Use% Mounted on\\n/dev/sda3        18G  2.0G   15G  12% /\\ntmpfs           931M     0  931M   0% /dev/shm\\n/dev/sda1       190M   40M  141M  22% /boot'"
}

TASK [debug conditionals filter] ***************************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The run commands status is changed"
}

TASK [debug int capitalize filter] *************************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The int value 1 The lower value is String"
}

TASK [debug default filter] ********************************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The variable value is ansible is not define"
}

TASK [debug list max and min filter] ***********************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The list max value is 5 The list min value is 1"
}

TASK [debug random filter] *********************************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The list random value is 4 and generate a random value is 171"
}

TASK [debug join filter] ***********************************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The join filter value is 1+2+3+4+5"
}

TASK [debug replace and regex_replace filter] **************************************************************************************
ok: [192.168.120.63] => {
    "msg": "The replace value is sTring The regex_replace value is ing"
}
......
```

### playbook 内置变量

#### groups 和 group_names

groups 变量是一个全局变量，它会打印出 Inventory 文件里面的所有主机以及主机组信息，它返回一个json 字符串，我们可以直接把它当作一个变量使用 {{ groups }}格式进行调用。我们可以引用 {{ groups }} 字符串里面的数据，例如引用 centos6 组的host，使用 {{ groups['centos6'] }} 方式引用，它会返回一个主机的list 列表，group_names 变量会打印当前主机所在的groups 名称。如果没有定义会返回 ungrouped，它返回的也是一个组名称的list 列表。

#### hostvars

hostvars 是用来调用指定主机变量，需要传入主机信息，返回的结果也是一个JSON字符串，同意也可以直接引用 JSON 字符串内的指定信息。

#### inventory_hostname 和 inventory_hostname_short

inventory_hostname 变量是返回 Inventory 文件里面定义的主机名，inventory_hostname_short 会返回 Inventory 文件中主机名的第一部分。

#### play_hosts 和 inventory_dir

play_hosts 变量是用来返回当前 playbook 运行的主机信息，返回格式是主机list结构，inventory_dir 变量是返回当前 playbook 使用 Inventory 目录。

#### 使用举例

定义的 inventory 如下：

```shell
# grep ^inventory /etc/ansible/ansible.cfg 
inventory      = /etc/ansible/inventory
# cat /etc/ansible/inventory/centos6 
[centos6]
192.168.120.6[1:3]
[centos6:vars]
ansible_ssh_pass='redhat'
[ansible:children]
centos6
# cat /etc/ansible/inventory/hosts
192.168.120.61 ansible_ssh_pass='redhat'
192.168.120.62 ansible_ssh_pass='redhat'
192.168.120.63 ansible_ssh_pass='redhat'
```

定义jinjia2 模板：

```jinja2
# cat jinjia.j2 
=================
groups info
{{ groups }}
=================

=============centos6 groups info================
{% for host in groups['centos6'] %}
={{ hostvars[host]['inventory_hostname'] }} eth0 IP is {{ hostvars[host]['ansible_default_ipv4']['address'] }}
+{{ hostvars[host]['inventory_hostname'] }} groups is {{ group_names }}
_{{ hostvars[host]['inventory_hostname'] }} short is {{ inventory_hostname_short }}
*{{ play_hosts }}
@{{ inventory_dir }}
{% endfor %}
```

定义渲染Jinjia 模板的playbook：

```yaml
# cat jinjia2.yaml 
- name: display jinjia2 template
  hosts: all
  tasks:
    - name: test template
      template: src=jinjia.j2 dest=/tmp/cpis
```

执行playbook：

```shell
# ansible-playbook jinjia2.yaml 

PLAY [display jinjia2 template] ****************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [192.168.120.61]
ok: [192.168.120.62]
ok: [192.168.120.63]

TASK [test template] ***************************************************************************************************************
changed: [192.168.120.62]
changed: [192.168.120.61]
changed: [192.168.120.63]

PLAY RECAP *************************************************************************************************************************
192.168.120.61             : ok=2    changed=1    unreachable=0    failed=0   
192.168.120.62             : ok=2    changed=1    unreachable=0    failed=0   
192.168.120.63             : ok=2    changed=1    unreachable=0    failed=0 
```

查看执行结果：

```shell
# cat /tmp/cpis 
=================
groups info
{'ungrouped': [], u'centos6': [u'192.168.120.61', u'192.168.120.62', u'192.168.120.63'], 'all': [u'192.168.120.61', u'192.168.120.62', u'192.168.120.63'], u'ansible': [u'192.168.120.61', u'192.168.120.62', u'192.168.120.63']}
=================

=============centos6 groups info================
=192.168.120.61 eth0 IP is 192.168.120.61
+192.168.120.61 groups is [u'ansible', u'centos6']
_192.168.120.61 short is 192
*[u'192.168.120.61', u'192.168.120.62', u'192.168.120.63']
@/etc/ansible/inventory
=192.168.120.62 eth0 IP is 192.168.120.62
+192.168.120.62 groups is [u'ansible', u'centos6']
_192.168.120.62 short is 192
*[u'192.168.120.61', u'192.168.120.62', u'192.168.120.63']
@/etc/ansible/inventory
=192.168.120.63 eth0 IP is 192.168.120.63
+192.168.120.63 groups is [u'ansible', u'centos6']
_192.168.120.63 short is 192
*[u'192.168.120.61', u'192.168.120.62', u'192.168.120.63']
@/etc/ansible/inventory
```

















