---
layout: post
title: "在 Fedora CoreOS 中运行容器"
date: 2023-02-20
tag: cloud-native
---

![Fedoraject-Anaconda-installer](/assets/images/2023-02-20/welcomefedoracoreos.jpg)

*[图片来源：[Fedora Docs](https://docs.fedoraproject.org/en-US/fedora-coreos/_images/welcomefedoracoreos.jpg) \]*

*作者: 邢万里*


Fedora CoreOS is an automatically updating, minimal, monolithic, container-focused operating system, designed for clusters but also operable standalone, optimized for Kubernetes but also great without it. It aims to combine the best of both CoreOS Container Linux and Fedora Atomic Host, integrating technology like Ignition from Container Linux with rpm-ostree and SELinux hardening from Project Atomic. Its goal is to provide the best container host to run containerized workloads securely and at scale.  
*🪧摘自 [Fedora CoreOS Documentation](https://docs.fedoraproject.org/jp/fedora-coreos/)*

# 写在前面

随着云原生的快速发展，将业务系统部署以 **Kubernetes** 这种容器编排系统为主的用户越来越多，在使用过程中，底层操作系统由于无法实现一致性配置而导致的问题频频出现，因此一个专门用于运行容器，使用了容器的设计理念的操作系统应运而生，常见的容器操作系统有 **Fedora CoreOS** 、 **Red Hat CoreOS** 、**RancherOS** 、**Photon** 、 **KubeOS** 等

本文使用 **Fedora CoreOS** 作为样例运行一个简单的容器。

*扩展知识*： **Fedora** 在早期有一个专门运行容器的产品 **Atomic** ，后来红帽将 **CoreOS** 收购之后，红帽将 **Atomic** 和 **CoreOS** 合并为 **Fedora CoreOS** ，简称 *fcos*，商业版本为**Red Hat CoreOS**，简称*RHCOS*。

# 准备工作

您需要事先准备一台和将来部署 **FCOS** 能够通信的一个 web 服务器，用用于存放 **ignition** 文件。

- 我这里使用的一台桌面版本 Linux 发行版：**Linux Mint 21.1**

![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-29-31.png)

- 网络信息如下

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ ip addr show ens37
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:d9:67:b5 brd ff:ff:ff:ff:ff:ff
    altname enp2s5
    inet 172.25.254.228/24 brd 172.25.254.255 scope global dynamic noprefixroute ens37
       valid_lft 155sec preferred_lft 155sec
    inet6 fe80::712d:e12e:ba61:7545/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```


- 并且我已经准备好了 Web 服务器

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ sudo systemctl status apache2.service
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2023-02-19 22:33:33 MST; 1h 58min ago
       Docs: https://httpd.apache.org/docs/2.4/
    Process: 947 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/SUCCESS)
    Process: 5087 ExecReload=/usr/sbin/apachectl graceful (code=exited, status=0/SUCCESS)
   Main PID: 1031 (apache2)
      Tasks: 55 (limit: 2182)
     Memory: 6.5M
        CPU: 289ms
     CGroup: /system.slice/apache2.service
             ├─1031 /usr/sbin/apache2 -k start
             ├─5105 /usr/sbin/apache2 -k start
             └─5106 /usr/sbin/apache2 -k start

Feb 19 22:33:31 workstation.lab.example.net systemd[1]: Starting The Apache HTTP Server...
Feb 19 22:33:33 workstation.lab.example.net systemd[1]: Started The Apache HTTP Server.
Feb 20 00:00:00 workstation.lab.example.net systemd[1]: Reloading The Apache HTTP Server...
Feb 20 00:00:00 workstation.lab.example.net systemd[1]: Reloaded The Apache HTTP Server.


┌💁  devops @ 💻  workstation in 📁  ~
└❯ ss -tplna | grep :8080
LISTEN 0      511                 *:8080               *:*           
```

- 请事先下载 **FCOS** 镜像

  - 最新版本Fedora下载站点 [https://getfedora.org/](https://getfedora.org/)

  - 本文使用镜像的链接 [https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/37.20230122.3.0/x86_64/fedora-coreos-37.20230122.3.0-live.x86_64.iso](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/37.20230122.3.0/x86_64/fedora-coreos-37.20230122.3.0-live.x86_64.iso)



### 1. 在**LinuxMint**中安装**butane**

**Fedora**社区为 Linux 客户端提供了两种方式来安装**butane**工具，这种工具可以将编写好的`ignition`文件转为 JSON 格式；第一种使用 **dnf** 进行安装，第二种使用 **brew**安装；我们采用 **brew**方式安装

- 首先安装 **brew** 工具

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

- 配置 **brew** 工具

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ (echo; echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"') >> /home/devops/.bashrc

┌💁  devops @ 💻  workstation in 📁  ~
└❯  sudo apt-get install build-essential

┌💁  devops @ 💻  workstation in 📁  ~
└❯  source ~/.bashrc

┌💁  devops @ 💻  workstation in 📁  ~
└❯  brew install gcc
```

- 使用 **brew**安装 **butane**

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯  brew install butane
```

- 生成密钥对

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯  ssh-keygen -t rsa -N '' -b 2048 -f ~/.ssh/id_rsa -q

┌💁  devops @ 💻  workstation in 📁  ~
└❯ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCms5d2FmXtXmhSJIezA812ymi4uT3XELM508FkD/4Pv+wXE0gl/aN7bPCVD6qxpHbXHgj6nWTIoej3pZeEYEOGlXua98tSyu3aaBfj5UIY5eF8838EireSJSVZPj3k6aB8ryL5W4G/4buwW6IzvvRRrknJDsFvajs5LvLiLUNrag/i9PIQZ0VZZQbLhx0/ZJCDAFIREpXuyPM+FsS2TYOnRZNeD/2IFscI05Xe4mA0NdI8Fzfji0c0dIBKYDD3mKFQ7qhvaLD7Q2v6I9hyfe5nZqAcaPW2f+rfsycdkhSOCZNQ7DbfWm7ILbAz+FFAUuzrMseytoEfOXbemQpgM7q5hIJgZkLOb38o4e6uXH+A43svE5U1tWKpaX2yRDSvi4rP9Nae0SdrkBzFG9dPzQvbEFYla+6BiFh+iRnnOLegni1WT9GjOonKay0/XCxH9s8mlXK1hKd/jgWf9j/BTkAOb++NZDCV5fsfGG+wnMUAcc44U6MNij+w3n4u6BiqAB0= devops@workstation.lab.example.net
```


### 2. 查看FCOS节点网络、硬件等信息

- 创建一台虚拟机，设置为 *UEFI* ，使用 **fcos** 光盘引导,界面如下

![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-51-53.png)

- 查看网卡名

![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-53-04.png)

- 如果您的虚拟机没有通过**DHCP**获取地址,需要手动设置网络信息，确保可以和提供`ignition`文件的主机进行通信

  - 设置网络的命令为 `sudo nmtui`

  ![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-55-12.png)

  ---
   ![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-55-42.png)

  ---
   ![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-56-21.png)

  ---

  - 激活网络连接

  ![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-57-04.png)
  
  ---
  - 使状态由 `Deactivate` 变为 `Activate`，再变为 `Deactivate`
  ![](/assets/images/2023-02-20/Snipaste_2023-02-20_15-58-00.png)

  - 检查是否生效

  ![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-05-57.png)

- 检查磁盘信息

![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-09-19.png)


### 3. 开始编写`ignition`文件

- 创建文件 `fcos36.yml`， 根据需要编写的`ignition`文件如下：

```yaml
variant: fcos
version: 1.4.0
# 创建用户 core 并使用 ssh 认证登录
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCms5d2FmXtXmhSJIezA812ymi4uT3XELM508FkD/4Pv+wXE0gl/aN7bPCVD6qxpHbXHgj6nWTIoej3pZeEYEOGlXua98tSyu3aaBfj5UIY5eF8838EireSJSVZPj3k6aB8ryL5W4G/4buwW6IzvvRRrknJDsFvajs5LvLiLUNrag/i9PIQZ0VZZQbLhx0/ZJCDAFIREpXuyPM+FsS2TYOnRZNeD/2IFscI05Xe4mA0NdI8Fzfji0c0dIBKYDD3mKFQ7qhvaLD7Q2v6I9hyfe5nZqAcaPW2f+rfsycdkhSOCZNQ7DbfWm7ILbAz+FFAUuzrMseytoEfOXbemQpgM7q5hIJgZkLOb38o4e6uXH+A43svE5U1tWKpaX2yRDSvi4rP9Nae0SdrkBzFG9dPzQvbEFYla+6BiFh+iRnnOLegni1WT9GjOonKay0/XCxH9s8mlXK1hKd/jgWf9j/BTkAOb++NZDCV5fsfGG+wnMUAcc44U6MNij+w3n4u6BiqAB0=

storage:
# 创建系统的根分区为 8GB
  disks:
    - device: /dev/disk/by-id/coreos-boot-disk
      wipe_table: false
      partitions:
        - number: 4
          label: root
          size_mib: 8192
          resize: true
        - size_mib: 0
          label: var
# 配置网卡 ens160 的网络信息
  files:
    - path: /etc/NetworkManager/system-connections/ens160.nmconnection
      mode: 0600
      contents:
        inline: |
          [connection]
          id=ens160
          type=ethernet
          interface-name=ens160
          [ipv4]
          address1=172.25.254.221/24;172.25.254.254
          dns=8.8.8.8;
          dns-search=lab.example.net
          may-fail=false
          method=manual
# 设置主机名
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: fcos.xwanli0923.github.io
# 创建容器本地持久化存储位置 
  directories:
    - path: /opt/website/html
      overwrite: true

# 创建容器所需的文件
    - path: /opt/website/html/index.html
      overwrite: true
      contents:
        inline: <h1>Hello, I am Xing Wanli. Welcome to cloudshelledu!</h1>
      mode: 0644
# 更改时区
  links:
    - path: /etc/localtime
      target: ../usr/share/zoneinfo/Asia/Shanghai
# 创建指定的文件系统
  filesystems:
    - path: /var
      device: /dev/disk/by-partlabel/var
      format: ext4
      with_mount_unit: true
# 创建一个 Systemd Unit 文件，用于开机运行 podman 容器，容器名为 mywebserver ，映射主机端口为 80 
systemd:
  units:
    - name: container-mywebserver.service
      enabled: true
      contents: |
        [Unit]
        Description=Podman container-mywebserver.service
        Documentation=man:podman-generate-systemd(1)
        Wants=network-online.target
        After=network-online.target
        RequiresMountsFor=%t/containers

        [Service]
        Environment=PODMAN_SYSTEMD_UNIT=%n
        Restart=on-failure
        TimeoutStopSec=70
        ExecStartPre=/bin/rm -f %t/%n.ctr-id
        ExecStart=/usr/bin/podman run --cidfile=%t/%n.ctr-id --sdnotify=conmon --cgroups=no-conmon --rm --replace -d --name mywebserver -v /opt/website/html:/usr/local/apache2/htdocs:Z -p 80:80 docker.io/library/httpd:latest
        ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id
        ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile=%t/%n.ctr-id
        Type=notify
        NotifyAccess=all

        [Install]
        WantedBy=multi-user.target default.target
```

- 将`ignition`文件转为 `ign` 格式

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ butane --pretty --strict fcos36.yml --output fcos36.ign 
```

- 将`ignition`文件存放在 Web 站点对应的目录下

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ sudo cp fcos36.ign /var/www/html/ign/
```

- 测试文件是否能访问

![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-19-12.png)

### 4. 使用`ignition`文件开始部署 FCOS

- 在FCOS的虚拟机中执行命令

```shell
[core@localhost ~]$ exit    # 如果看不到安装帮助命令，可先执行 `exit` 
[core@localhost ~]$ sudo coreos-installer install /dev/sda \
                    --ignition-url http://172.25.254.228:8080/ign/fcos36.ign \
                    --insecure-ignition
```
---
![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-22-44.png)

---
- 安装结束，重启系统

![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-27-07.png)

- 登录界面

![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-29-17.png)


### 5. 远程连接并验证结果

- 使用 SSH 私钥进行连接

```shell
┌💁  devops @ 💻  workstation in 📁  ~
└❯ ssh -i ~/.ssh/id_rsa core@172.25.254.221
The authenticity of host '172.25.254.221 (172.25.254.221)' can't be established.
ED25519 key fingerprint is SHA256:RmfcSQncM52Z6pn8acIrpXjsLpw167fDyiSjVKHCy0I.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.25.254.221' (ED25519) to the list of known hosts.
Fedora CoreOS 37.20230122.3.0
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

[core@fcos ~]$ 
```

- 查看磁盘分区

```shell
[core@fcos ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0    1M  0 part 
├─sda2   8:2    0  127M  0 part 
├─sda3   8:3    0  384M  0 part /boot
├─sda4   8:4    0    8G  0 part /sysroot/ostree/deploy/fedora-coreos/var
│                               /usr
│                               /etc
│                               /
│                               /sysroot
└─sda5   8:5    0 11.5G  0 part /var/lib/containers/storage/overlay
                                /var
```

- 查看网络配置

```shell
[core@fcos ~]$ hostname
fcos.xwanli0923.github.io
[core@fcos ~]$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:d0:c3:a7 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 172.25.254.221/24 brd 172.25.254.255 scope global dynamic noprefixroute ens160
       valid_lft 151sec preferred_lft 151sec
    inet6 fe80::6f14:2875:e850:8fd9/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: podman0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ce:16:23:ac:2f:1d brd ff:ff:ff:ff:ff:ff
    inet 10.88.0.1/16 brd 10.88.255.255 scope global podman0
       valid_lft forever preferred_lft forever
    inet6 fe80::9c82:7aff:fec2:5025/64 scope link 
       valid_lft forever preferred_lft forever
4: veth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master podman0 state UP group default qlen 1000
    link/ether 32:35:08:f9:a6:c3 brd ff:ff:ff:ff:ff:ff link-netns netns-a5949274-3549-2bb7-7ce9-36465802458e
    inet6 fe80::3035:8ff:fef9:a6c3/64 scope link 
       valid_lft forever preferred_lft forever
```

- 检查容器

```shell
[core@fcos ~]$ sudo podman ps 
CONTAINER ID  IMAGE                           COMMAND           CREATED        STATUS            PORTS               NAMES
3cec19adf88f  docker.io/library/httpd:latest  httpd-foreground  6 minutes ago  Up 6 minutes ago  0.0.0.0:80->80/tcp  mywebserver
```

- 客户端访问

![](/assets/images/2023-02-20/Snipaste_2023-02-20_16-39-59.png)

---

*参考文档：*
*- [Fedora CoreOS裸金属安装](https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/)*
