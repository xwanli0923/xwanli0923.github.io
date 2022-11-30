---
layout: post
title: "如何卸载 Ansible Tower"
date: 2022-11-29
tag: ansible
---

![Ansible](https://nwzimg.wezhan.cn/contents/sitefiles2033/10168677/images/38319670.jpg)

* 使用 ` ansible-tower-service stop` 命令停止 Ansible 相关服务，并使用 `ansible-tower-service status`  命令确保服务已经停止成功。
```bash
$ ansible-tower-service stop
Stopping Tower
Redirecting to /bin/systemctl stop rh-postgresql10-postgresql.service
Redirecting to /bin/systemctl stop rabbitmq-server.service
Redirecting to /bin/systemctl stop nginx.service
Redirecting to /bin/systemctl stop supervisord.service
```
* 将 `/etc/ansible` 和 `/etc/tower` 目录进行归档。
```bash
$ tar -cf ansible.tar /etc/ansible
$ tar -cf tower.tar   /etc/tower
```
* 另外，假设您已经使用 `setup.sh` 安装了 Ansible Tower,你可以使用以下命令进行备份 Ansible Tower。这样你可以在重新安装 Ansible Tower 后使用 `setup.sh -r <backup file>` 命令进行还原。
```bash
$ setup.sh -b
```
* 使用 `yum remove` 命令卸载 `Ansible`，`RabbitMQ` 和 `Python`。
```bash
$ yum remove ansible-tower*
$ yum remove rabbitmq-server
$ yum remove rh-python36-*
```
* 删除 `Postgres` 脚本。
```bash
$ rm /etc/profile.d/rh-postgresql10-env.sh
```
* 删除以下目录。
```bash
$ rm -rf /etc/ansible
$ rm -rf /etc/tower
$ rm -rf /var/lib/pgsql
$ rm -rf /var/lib/awx
$ rm -rf /var/lib/rabbitmq
$ rm -rf /var/opt/rh/rh-postgresql10/lib/pgsql/data
```
* 使用 `yum clean` 命令清除 `Ansible Tower` 软件仓库
```bash
$ yum clean metadata --enablerepo="ansible-tower,ansible-tower-dependencies"
```

* 确保 `rpm` 命令没有输出
```bash
$ rpm -qa | grep ansible-tower
```
---
> 来源：[http://www.freekb.net/Article?id=212](http://www.freekb.net/Article?id=212)   
> 翻译： 邢万里
