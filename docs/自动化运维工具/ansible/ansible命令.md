

## 配置文件简介

forks = 5

gathering = implicit

host_key_checking = False

log_path = /var/log/ansible.log

inventory= /etc/ansible/hosts

module_name = command

remote_port = 22

roles_path= /etc/ansible/roles

timeout = 10

[详细参考]( https://ansible-tran.readthedocs.io/en/latest/docs/intro_configuration.html )





## Inventory



## Variables



## 模块

参考模块目录



## playbook

tasks

handlers

tags

vars：Inventory 文件中、 facts、通过命令行指定、playbook 定义、独立的yaml 文件指定、role中定义

template  for 循环 if 判断

when

with_items



## roles

