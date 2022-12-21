---
layout: post
title: "部署CEPH集群前的准备工作"
date: 2022-12-19
tag: ceph
---

![Ceph](/assets/images/2022-12-19/photo-squid-02.jpg)
*\[图片来源 ceph.io\]*

## 文档说明

- 系统版本：CentOS-Stream-8-x86_64-20221125

- 内核版本：4.18.0-408

- podman 版本:4.2.0

- ceph 版本： Quincy

- 其他要求： **git**、**LVM2**、**python3.6.8**

- 客户端配置：
  - **ceph-clienta**
    - CPU: 2vCPU
    - Memory: 4GiB
    - Storage: 10GiBx5

  - **ceph-clientb**
    - CPU: 2vCPU
    - Memory: 4GiB

*ceph-clienta 充当ceph集群客户端的同时，也充当ceph的管理端*

- 集群节点配置：

  - **ceph-serverc**、**ceph-serverd**和**ceph-servere**
    - CPU: 2vCPU
    - Memory: 4GiB
    - Storage: 10GiBx5

- 网络信息：
  **public_network**: 172.16.80.0/24
  **cluster_network**: 172.16.90.0/24    

## 安装 CEPHADM 

1. 使用`root`账户登录到`ceph-serverc`节点

```shell
$ dnf install -y centos-release-ceph-quincy
$ dnf install -y cephadm vim bash-completion git ansible
```

## 设置系统

1. 在 `clientc`创建`admin`账户，并设置其为免认证`SUDO`

```shell
$ useradd admin
$ passwd admin
$ echo "admin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/admin
$ chmod 0400 /etc/sudoers.d/admin
```

2. 在 `clientc`上创建免密码认证的 ssh 密钥对，并复制到`admin`家目录

```shell
$ ssh-keygen -q -t rsa -f ~/.ssh/id_rsa -N ''
$ cp -r ~/.ssh  ~admin/.ssh
$ chown -R admin: ~admin/.ssh
$ ssh-copy-id root@localhost
```

3. 根据自己的情况，编写本地主机名解析文件 `/etc/hosts`,追加以下内容

```
172.16.80.102 ceph-clienta.lab.example.net ceph-clienta
172.16.80.103 ceph-clientb.lab.example.net ceph-clientb
172.16.80.104 ceph-serverc.lab.example.net ceph-serverc
172.16.80.105 ceph-serverd.lab.example.net ceph-serverd
172.16.80.106 ceph-servere.lab.example.net ceph-servere
```

4. 将上述创建的用户，文件等在其他节点同样创建

```shell
#!/bin/bash
for HOSTS in ceph-{clienta,clientb,serverc,serverd,servere}
  do
    scp -r /root/.ssh root@${HOSTS}:/root/.ssh
    scp  /etc/hosts root@{HOSTS}:/etc/hosts
    ssh root@${HOSTS} "useradd admin"
    ssh root@${HOSTS} "echo YOUR_ADMIN_PASSWORD | passwd --stdin admin"
    scp /etc/sudoers.d/admin root@${HOSTS}:/etc/sudoers.d/admin
    ssh root@{HOSTS} "chmod 0400 /etc/sudoers.d/admin"
    scp -r /root/.ssh root@${HOSTS}:/home/admin/.ssh
    ssh root@{HOSTS} "chown -R admin: /home/admin/.ssh"
  done
```

## 准备 cephadm-ansible

1. 使用 `git clone` 命令下载 `cephadm-ansible`

```shell
git clone -b quincy https://github.com/ceph/cephadm-ansible.git
cd cephadm-ansible/
```

2. 在 `cephadm-adnible` 下创建 `hosts`清单文件
```ini
ceph-clienta.lab.example.net
ceph-clientb.lab.example.net
ceph-serverc.lab.example.net
ceph-serverd.lab.example.net
ceph-servere.lab.example.net

[clients]
ceph-clienta.lab.example.net
ceph-clientb.lab.example.net

[admin]
ceph-clienta.lab.example.net
```
8. 测试清单中的主机，并执行 `cephadm-preflight.yml`:
   
   - 测试主机的连通性和用户
   ```shell
   $ ansible -i hosts --list-hosts all
     hosts (4):
        ceph-serverc.lab.example.net
        ceph-serverd.lab.example.net
        ceph-servere.lab.example.net
        ceph-clienta.lab.example.net
        ceph-clientb.lab.example.net
   
   $ ansible -i hosts -m ping all
   $  ansible all -i hosts  -u admin -b -m ping
   ```
  
  - 查看 `cephadm-preflight.yml`
  
    ```yaml
    # Usage:
    #
    # ansible-playbook -i <inventory host file> cephadm-preflight.yml
    #
    # You can limit the execution to a set of hosts by using `--limit` option:
    #
    # ansible-playbook -i <inventory host file> cephadm-preflight.yml --limit <my_osd_group|my_node_name>
    #
    # You can override variables using `--extra-vars` parameter:
    #
    # ansible-playbook -i <inventory host file> cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"
    #
    ```

  - 使用 `--extra-vars "ceph_origin=community" 执行 `cephadm-preflight.yml`
    ```shell
    $  ansible-playbook -i hosts --extra-vars "ceph_origin=community" cephadm-preflight.yml
    ```

## 设置默认登录账户(可选)

1. 在 `root`和`admin` 的 `$HOME/.ssh/config`中增加以下内容(没有则创建该文件)：
```ini
Host *.lab.example.net
     User root
     StrictHostKeyChecking no
     UserKnownHostsFile /dev/null
```
  
2. 将上述文件分别复制到其他节点的对应用户家目录中：
```shell
#!/bin/bash
for HOSTS in ceph-{clienta,clientb,serverc,serverd,servere}
    do
       scp /root/.ssh/config root@${HOSTS}:~/.ssh/config
       scp /home/admin/.ssh/config admin@${HOSTS}:~/.ssh/config
       ssh root@{HOSTS} "systemctl disable firewalld.service --now"
    done
```   
---
> 参考文档：
> Ceph  Document: [distribution-specific-installations](https://docs.ceph.com/en/latest/cephadm/install/#distribution-specific-installations)
> cephadm-ansible Document: [https://github.com/ceph/cephadm-ansible](https://github.com/ceph/cephadm-ansible/blob/devel/README.md)